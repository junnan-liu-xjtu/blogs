# Gradle Build Optimization: from 5 Minutes to 1 Minute

This article talks about the optimization process of gradle build on the continuous integration (CI) pipelines, for a server-side program written in Java/Kotlin + Spring Boot.

## Gradle build takes too long on CI agents

Since the CI agents that execute the pipeline scripts are shared by several teams, and the technology stacks used by each service are different, the CI agents are expected to be able to build different kinds of programs. For example, some projects require Java, while others require NodeJS. Even if several projects all require Java, they might require different versions of Java on CI agents. In order to avoid repeated installation of different versions of Java on each CI agent, and to ensure that the Java version used in the CI tests is consistent with the one in the deployed environment, we use the docker container as the runtime for gradle build, so that as long as the CI agent has docker, it can run different builds on the exact version.

The simplest and the most straightforward command is as follows:

```shell
docker run -v $(pwd):/app -w /app eclipse-temurin:11-jre ./gradlew clean build
```

If you really use this command to run every pipeline, you will find that it is excruciatingly slow, because almost an empty Spring Boot repo can take 5 minutes to finish the build step. After adding a debugging parameter `-i` to the above gradle command, it is not difficult to find in the logs that a lot of time is spent on downloading dependencies. And because gradle build is run in the new container every time, the dependencies downloaded by the previous build cannot be reused by the latter one.

## Caching the Downloaded Dependencies

### Attempt 1: Utilize Docker Build Cache

In order to reuse the dependencies, the first thing comes to our minds is [docker build cache](https://docs.docker.com/build/building/cache/). Long story short, docker images are built layer by layer, and if layers didn’t change, docker will not build them over and over again, they can be cached. The docker official website provides a [NodeJS example](https://docs.docker.com/build/building/cache/#order-your-layers). The main idea is that, on the first step, only copy `package.json` and `yarn.lock`, which define the dependencies and have a low frequency of change. The next step is `npm install`, pulling all the dependencies defined by the previous layer, and after that, copy all the business code, which has a high frequency of change, then run `npm build`. In this way, as long as the dependency definition files are not changed, the layer with installed dependencies are reused.

We concluded a similar solution on our Java build:

```dockerfile
FROM eclipse-temurin:11-jre as builder
WORKDIR /app/
COPY build.gradle.kts settings.gradle.kts gradle.properties gradlew ./
COPY gradle ./gradle
RUN ./gradlew clean build
COPY . .
RUN ./gradlew clean build

FROM eclipse-temurin:11-jre as app
WORKDIR /app/
COPY --from=builder /app/build/libs/*.jar .
COPY ./entrypoint.sh .

ENTRYPOINT ["./entrypoint.sh"]

CMD [""]
```

In the same way, we copied the dependency definition files first, then installed the dependencies, and then copied our business code, and then built our artifact, to save the time of repeatedly downloading dependencies. With the help of `.dockerignore`, we could exclude all the files that are irrelevant to the jar file. As a result, if there are no business code changes, we can reuse all the layers and the build will be incredibly fast.

In the above solution, we also utilized the [multi-stage](https://docs.docker.com/build/building/multi-stage/) build of docker, to reduce the size of the image, ultimately to reduce the deployment time on pulling the image.

After this change, the build step in the pipeline triggered by pure business code changes, on the CI agents where cache is available, took only 1 minute 50 seconds. From the logs we could confirm that the layers before copying the business code were using the cache.

```shell
[14:45:03] $ docker build .
[14:45:03] Sending build context to Docker daemon  403.5kB
[14:45:03] Step 1/13 : FROM eclipse-temurin:11-jre as builder
[14:45:03]  ---> 6c1d7cfd8f3c
[14:45:03] Step 2/13 : WORKDIR /app/
[14:45:03]  ---> Using cache
[14:45:03]  ---> 34bd2aa46830
[14:45:03] Step 3/13 : COPY build.gradle.kts settings.gradle.kts gradle.properties gradlew ./
[14:45:03]  ---> Using cache
[14:45:03]  ---> 4e2748bad63f
[14:45:03] Step 4/13 : COPY gradle ./gradle
[14:45:03]  ---> Using cache
[14:45:03]  ---> 831fd4b1b6ff
[14:45:03] Step 5/13 : RUN ./gradlew clean build
[14:45:03]  ---> Using cache
[14:45:03]  ---> 9b08d16abb7e
[14:45:03] Step 6/13 : COPY . .
[14:45:03]  ---> 40b37551d8a7
[14:45:03] Step 7/13 : RUN ./gradlew -w clean build
[14:45:03]  ---> Running in e24187b324eb
```

However, those benefits happened under a lot of conditions. The main one is that the CI agents are required to have recently executed the same pipeline. If the repo is not in a very active state, we can rarely take advantage of the improvement, because the agents are usually cleaned up or replaced periodically. Without cache, the pipeline will still need 5 minutes.

### Some Possible Improvements on Attempt 1

Reusing caches across the CI agents is not impossible. [Exporting docker build cache](https://docs.docker.com/engine/reference/commandline/buildx_build/#cache-to) is one of the solutions. A lot of pipeline services can pass artifacts across different agents. Besides, uploading to and downloading from a shared place like a static server or AWS S3 bucket is also a feasible solution. Pushing the intermediate image to the image registry and pulling it before the next build might be also a good idea. As long as these costs are lower than downloading the fresh dependencies, they are good solutions.

### Attempt 2: Sharing Gradle Dependency Cache

Now that we are able to pass caches among CI agents, why not pass [gradle dependency cache](https://docs.gradle.org/current/userguide/dependency_resolution.html#sec:dependency_cache) directly? Compared to docker build cache, which will be invalidated on any changes to the file content, gradle dependency cache appears to be more intellectual. Gradle has its logic to decide whether the dependencies can be reused. With gradle dependency cache, adding a new dependency in `build.gradle` file will trigger gradle downloading only the new dependency, while with docker build cache, because it’s checking the file change and invalidating the whole layer, it will cause the downloading of all the dependencies.

With gradle caches on CI agents, all we need to do is to mount the gradle cache to the build container.

```shell
docker run -v $(pwd):/app -v "$HOME/.gradle:/root/.gradle" -w /app eclipse-temurin:11-jre ./gradlew clean build
```

By this solution, our gradle build time came to 1 minute 7 seconds. Adding the average time of 15 seconds for uploading and downloading the caches, we got the 1 minute 22 seconds total time, which is slightly faster than docker build cache. A more important benefit of this solution is that we can reuse the downloaded caches as much as possible for almost every build, which will be much more efficient than leveraging docker build cache.

In this way, if we reformat some of our business code, all the caches are fully reused, which results in a super fast build. Even when we added a new dependency, the new build only took 1 minute 51 seconds in total, which was only slightly slower than the fully cached version. Compared to invalidating all the build layers, this is far more efficient.

## Eliminating Unnecessary Gradle Tasks

Although we have reduced our build time by caching the dependencies, the nearly empty project still needs more than 1 minute to build. It only includes 3 test classes with 10 test cases. As the project gets larger, the build time will increase rapidly. To further decrease the build time, we started the analysis on gradle tasks. Gradle provides a [profile report](https://docs.gradle.org/current/userguide/performance.html#profile_report) feature, which can display the detailed time consumption on each task. We executed `./gradlew clean build --profile` locally, and got the result as below


| Task                            | Duration | Result      |
|---------------------------------|----------|-------------|
| :                               | 26.870s  | (total)     |
| :test                           | 16.078s  |             |
| :compileKotlin                  | 2.087s   |             |
| :distZip                        | 1.875s   |             |
| :compileTestKotlin              | 1.735s   |             |
| :bootDistZip                    | 1.518s   |             |
| :bootStartScripts               | 1.037s   |             |
| :jacocoTestReport               | 0.852s   |             |
| :bootJar                        | 0.693s   |             |
| :distTar                        | 0.254s   |             |
| :bootDistTar                    | 0.249s   |             |
| :bootJarMainClassName           | 0.118s   |             |
| :clean                          | 0.084s   |             |
| :lintKotlinMain                 | 0.080s   |             |
| :jacocoTestCoverageVerification | 0.078s   |             |
| :startScripts                   | 0.062s   |             |
| :lintKotlinTest                 | 0.055s   |             |
| :processResources               | 0.006s   |             |
| :inspectClassesForKotlinIC      | 0.004s   |             |
| :processTestResources           | 0.003s   |             |
| :compileJava                    | 0.001s   | NO-SOURCE   |
| :compileTestJava                | 0.001s   | NO-SOURCE   |
| :assemble                       | 0s       | Did No Work |
| :build                          | 0s       | Did No Work |
| :check                          | 0s       | Did No Work |
| :classes                        | 0s       | Did No Work |
| :jar                            | 0s       | SKIPPED     |
| :lintKotlin                     | 0s       | Did No Work |
| :testClasses                    | 0s       | Did No Work |

In the report we noticed a lot of tasks that we don’t need, such as distTar, `bootDistTar`, `distZip`, `bootDistZip`. After checking each task, we finally remained the following: `lintKotlin`, `test`, `jacocoTestCoverageVerification`, `jacocoTestReport`, `bootJar`. After eliminating the unnecessary tasks, our build step is 8 seconds faster. Moreover, checking the test report generation task is also important because removing the unnecessary report generation will help as well.


| Task                            | Duration | Result      |
|---------------------------------|----------|-------------|
| :                               | 18.644s  | (total)     |
| :test                           | 13.418s  |             |
| :compileKotlin                  | 1.866s   |             |
| :compileTestKotlin              | 1.494s   |             |
| :jacocoTestReport               | 0.752s   |             |
| :bootJar                        | 0.690s   |             |
| :bootJarMainClassName           | 0.141s   |             |
| :clean                          | 0.096s   |             |
| :jacocoTestCoverageVerification | 0.071s   |             |
| :lintKotlinTest                 | 0.054s   |             |
| :lintKotlinMain                 | 0.052s   |             |
| :processResources               | 0.005s   |             |
| :processTestResources           | 0.004s   |             |
| :compileTestJava                | 0.001s   | NO-SOURCE   |
| :classes                        | 0s       | Did No Work |
| :compileJava                    | 0s       | NO-SOURCE   |
| :lintKotlin                     | 0s       | Did No Work |
| :testClasses                    | 0s       | Did No Work |

## Improving Test Efficiency

Another optimization point for a general build step is the test execution. Real unit tests are very fast, while those requiring the startups of the whole framework, such as Spring, are slow, because the startup of the framework is slow. We call these tests as integration tests, as a comparison to unit tests. Actively maintaining the test quantity according to the test pyramid is the first and foremost step of optimizing the test efficiency. What we can do more is to reduce the start time of Spring by reusing the test application context.

Are you still satisfied with the sliced startup of Spring at the hands of `@WebMvcTest`, `@DataJpaTest`, thinking that these annotations speeded up your tests? Actually we can do better. If you have a glance at Spring Boot test [context caching](https://docs.spring.io/spring-framework/docs/current/reference/html/testing.html#testcontext-ctx-management-caching), you will realize that Spring tests are fairly clever. They will reuse the `ApplicationContext` if nothing changes on it in the previous test.

Then how can we maximize the benefits of context caching? The answer is to try to avoid changing the `ApplicationContext` by annotations such as `@MockBean`, `@SpyBean`, `@DirtiesContext`, `@TestPropertySource`, `@DynamicPropertySource`.

On this purpose, when checking our existing tests, we found we had `@SpringBootTest`, `@WebMvcTest`, and `@DataJpaTest`. When we executed the tests the logs show that Spring framework was started 3 times, which took a lot of time. In order to reuse `ApplicationContext`, we changed them to `@SpringBootTest` (change 1), to avoid restarting Spring during different tests. Then we found we had `@MockBean`. Our solution is to add the mocked bean to a test configuration file (change 2), so that they are the same for every integration test, and the `ApplicationContext` was kept the same in different tests. The test code are as follows.

Before change:

```kotlin
@WebMvcTest
class ATest(@Autowired val mvc: MockMvc) {
    @MockBean
    lateinit var bService: BService

    @Test
    fun blahblah() {
      //...
    }
}
```

After change:

```kotlin
@SpringBootTest  // change 1
@AutoConfigureMockMvc  // change 1
class ATest(
    @Autowired val mvc: MockMvc,
    @Autowired val bService: BService // change 2
) {
  @Test
  fun blahblah() {
    //...
  }
}

@Configuration // change 2
class TestConfiguration {
  @Bean
  fun bService(): BService {
    return Mockito.mock(BService::class.java)
  }
}
```

After the changes, we executed the tests. The logs showed that Spring was only started once, which was exactly what we were expecting. The profile report also proved our theory since the test time was reduced by 3 seconds. If you have a lot of integration tests and find it hard to keep them all in a same `ApplicationContext`, extracting a `@IntegrationTest` annotation and define all the context-related things in the annotation, and use the annotation in different tests, it will be easier to control.

After all these improvements, we pushed our code to execute it on the CI agent. Our gradle build step consumed only 58 seconds. Adding all the cache pushing and pulling, it took 1 minute 13 seconds.

## Conclusion

We reduced the CI build time on our gradle build step, from 5 minutes to 1 minute, mainly through 4 ways:
1. Caching the downloaded dependencies by mounting gradle cache.
2. Sharing cache files among CI agents to maximize the benefits of caching.
3. Eliminating unnecessary gradle tasks to reduce gradle build time.
4. Trying to reuse ApplicationContext in integration tests to reduce the startups of Spring framework.

Besides, gradle official website provided other tips about performance. For those who are concerned please refer to <https://docs.gradle.org/current/userguide/performance.html>.
