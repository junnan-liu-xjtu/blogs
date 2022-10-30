# gradle build 从 5 分钟到 1 分钟

本篇文章记录的是 java/kotlin + spring boot 的服务端项目，在持续集成(CI)流水线(pipeline)上执行 gradle build 的优化过程。

## 1. 时间超长的 gradle build

由于执行流水线的机器 (ci agent) 是几个团队共用的，而各服务用到的技术栈不尽相同，甚至同为 java 项目，java 版本也不同。为了避免在每一个 ci agent 上重复安装不同版本的 java，又想保证执行测试时所用的 java 版本与最终部署时所用的 java 运行时版本一致，我们使用 docker 容器作为打包运行时，进行 gradle build，于是使用最简单直白的指令如下：

```bash
docker run -v $(pwd):/app -w /app eclipse-temurin:11-jre ./gradlew clean build
```

如果你真的用这个指令跑每一次 ci，你就会发现它慢得令人发指，因为近乎一个空的 spring-boot repo，跑完 build 这一步骤都要花费5分钟。给上述 gradle 指令加上调试参数 `-i` 后，我们在日志中不难发现，大量的时间花费在了下载依赖上。而且由于每次都在容器中跑 gradle build，跑完以后的依赖并不能被下一次 build 重用。

## 2. 依赖下载缓存

#### 2.1 尝试一：利用 docker build cache

为了达到重用的目的，我们第一步想到了 [docker build cache](https://docs.docker.com/build/building/cache/)。简而言之，就是在构建 docker image 的时候是逐层构建，如果前面几层的文件和指令都相同，那么 docker 并不会做重复工作，而是会使用缓存。官网还举了一个 node 环境的[例子](https://docs.docker.com/build/building/cache/#order-your-layers)，为我们提供了思路。

优化前：

```Dockerfile
FROM node
WORKDIR /app
COPY . .          # Copy over all files in the current directory
RUN npm install   # Install dependencies
RUN npm build     # Run build
```

优化后

```Dockerfile
FROM node
WORKDIR /app
COPY package.json yarn.lock .    # Copy package management files
RUN npm install                  # Install dependencies
COPY . .                         # Copy over project files
RUN npm build                    # Run build
```

例中优化前的版本，由于易变更的业务代码过早引入，导致每次 npm install 都需要重新拉取依赖，非常的耗时。而优化后的 Dockerfile 通过单独拷贝两个不易变更的 package.json 和 yarn.lock 两个文件，而后先行安装依赖的方式，让依赖不变的情况下，前四层能够利用 docker build 的缓存，避免重复拉取依赖。然后才将易变更的业务代码拷贝，进行 build，这样一来，如果 ci agent 之前执行过当前 repo 的流水线，那么有很大概率能节省掉拉取依赖的时间。

道理懂了，我们如法炮制，构建出 dockerfile 如下：

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

道理也很简单，先把 gradle 自身的文件，以及其定义依赖的文件拷贝，然后先行安装依赖，让这些层最大限度地使用缓存，然后再引入业务代码，再进行 build，可以为我们节省很多时间。同时结合 `.dockerignore` 文件将与 build 不相干的文件全部排除，甚至可以在业务代码不变的情况下，连测试和打包都利用缓存，使 ci 更加迅速。然后利用分段构建，只取 jar 包和 base image，减小 image 体积，从而缩短部署时拉镜像的时长，进一步缩短 ci 耗时。

经此优化后，普通的业务代码变更，在有缓存的 agent 上，ci 只需要1分50秒，而且从日志可以看出，在业务代码之前的部分都使用了 cache。

```bash
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

当然，这样的收益，是有条件的，它要求 ci agent 近期执行过当前 repo 的流水线。如果这是一个不经常开发的 repo，可能几乎享受不到收益，因为 agent 可能会定期清理或更换。没有缓存的收益，这样的流水线执行仍然需要5分钟。

#### 2.2 转机：跨 agent 共享 cache

此时我们能想到的是，如果 build cache 能够被 pipeline 随身携带就好了，任何一个 agent 都有相关的 build cache，将大幅提升效率。幸运的是，[export docker build cache](https://docs.docker.com/engine/reference/commandline/buildx_build/#cache-to) 提供了这样的可能。

在 cache 导出以后，再通过各 pipeline 的互传 artifact 机制，或者利用可以上传和下载文件的插件，进行跨 agent 的传输即可实现。至于互传 cache 的指令或插件，它的责任很简单，就是在每个 pipeline step 执行前，将之前上传的 cache 文件夹下载到本地，并在 step 执行以后，将 cache 继续上传即可。上传和下载 cache 的目的地可能都是在内网，或在同一网络环境内，如果速度明显超过 gradle build 从 maven repository 下载依赖的速度，那么这种方法着实可行。需要注意的是，在 repo 较多的团队中，这些依赖可能要根据种类或 repo 进行区分，如当前 repo 有其专用的 cache，否则下载不相关的依赖也会消耗不必要的时间。

但是我们并没有按照这一方法继续定制化，因为我们得知，其他团队已经可以通过携带和挂载 gradle cache 的方式加速 ci，且大团队内部已经实现了互传 cache 的相关插件，我们只需要引用该插件即可。

#### 2.3 尝试二：携带和挂载 gradle dependency cache

对呀，既然都想到在各 agent 之间传递 cache 了，为什么不直接传递 [gradle dependency cache](https://docs.gradle.org/current/userguide/dependency_resolution.html#sec:dependency_cache) 呢？相比之下，docker build cache 会在上述几个依赖定义文件内容发生改变时失效，如 `build.gradle`，`gradle.properties` 等，添加一个依赖会导致整层缓存失效，所有依赖需要重新下载，甚至格式的改变也会如此。而利用 gradle dependency cache 的方式则更有优势，因为 gradle 会对其下载的依赖进行判断，来决定是重用还是需要下载新的依赖，让缓存发挥最大作用，从而减少不必要的时间消耗。

现在 ci agent 有了 gradle cache，只需要在 docker gradle build 时将 gradle cache 挂载到容器中即可。

```bash
docker run -v $(pwd):/app -v "$HOME/.gradle:/root/.gradle" -w /app eclipse-temurin:11-jre ./gradlew clean build
```

使用这种办法，我们 gradle build 的执行时间缩短到了1分7秒，加上平均15秒的上传和下载 cache 的时间，总体略快于上述利用 docker build cahce 的方法，但是这种方法可以使每次 ci 都充分利用缓存的便利，效率更高。在调整了 `build.gradle` 文件的格式以后，ci 执行时间几乎没有改变。甚至在我们添加了一个依赖以后，因为只需要下载新加的依赖，时间也仅仅是1分36秒，加上加上平均15秒的上传和下载 cache 的时间，也远远好于需要下载所有依赖的5分钟。

## 3. 精简 gradle tasks

关于缩短依赖拉取的时间，我们已经满意了。可是为什么还要1分钟多呢？明明只有3个测试类，10个测试用例呀！以后代码多了会不会成倍增长？为了进一步缩短 build 的时间，我们开始了对 gradle tasks 的分析。gradle 提供了[profile report](https://docs.gradle.org/current/userguide/performance.html#profile_report) 的功能，可以详细展示每一步工作的耗时情况。通过在本地执行 `./gradlew clean build --profile`，我们得到了下列结果（本小节都在本地对比）：

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

其中很明显有一些我们并不需要的，比如 `distTar`, `bootDistTar`, `distZip`, `bootDistZip` 等。通过仔细地对比和理解每一个 task 所做的工作，我们最终留下了 `lintKotlin`, `test`, `jacocoTestCoverageVerification`, `jacocoTestReport`, `bootJar`，通过在 `build.gradle` 中定义 task 之间的依赖，我们只需要执行 `./gradlew clean test bootJar` 就可以达到我们的期望了。同时精简之后本地的 profile report 如下，直接减少了不必要的 8 秒。PS：减少生成不必要的测试报告也能提供一点优化。

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

## 4. 提升测试速度

现在就剩下测试的执行了。真正的单元测试执行得很快，反观需要启动 spring 的集成测试，花在启动 spring boot 上的时间占了很大一部分。除了保证测试策略的正确性和按照测试金字塔安排测试数量以外，在相同测试数量的情况下尽量减少 spring boot 在整个测试中的启动时间，能大大优化测试的执行效率。

你还在喜于使用了 `@WebMvcTest`、`@DataJpaTest` 而减少了 spring context 的启动加载类的数量，从而减少了整体测试时间？你可能用错了哦。如果你了解过 spring boot test [context caching](https://docs.spring.io/spring-framework/docs/current/reference/html/testing.html#testcontext-ctx-management-caching)，你会发现 spring test 是足够聪明的，如果 ApplicationContext 没有发生改变，在执行下一个需要启动 spring 的测试类时，spring 会重用原来的上下文。

那么怎样能让我们尽可能地受益于 context caching？其实就是要尽可能地避免让测试觉得 ApplicationContext 变过，如避免 `@MockBean`, `@SpyBean`, `@DirtiesContext`, `@TestPropertySource`, `@DynamicPropertySource` 等等。

再次审查我们的测试，其中有 `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`，执行一下，发现 spring 启动了 3 次，比较浪费时间。本着尽可能共用 ApplicationContext 的原则，把他们都换成 `@SpringBootTest` (change 1)，毕竟另外两个就算包含的 bean 再少，也不如直接不重新启动 spring 来的实在。此外我们还发现测试中有 `@MockBean` 存在，优化的方式是将他们以 bean 的形式放到测试的 `@Configuration` 中 (change 2)，供所有测试使用，来保证上下文的一致性。以下为示例代码：

优化前：

```kotlin
@WebMvcTest
class ATest(@Autowired val mvc: MockMvc) {
    @MockBean
    lateinit var bService: BService
 
    @Test
    fun `blahblah`() {
        ...
    }
}
```

优化后

```kotlin
@SpringBootTest  // change 1
@AutoConfigureMockMvc  // change 1
class ATest(
    @Autowired val mvc: MockMvc,
    @Autowired val bService: BService // change 2
) {
    @Test
    fun `blahblah`() {
        ...
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

改好以后，通过观察测试日志，发现在执行所有10个测试的过程中， spring 只启动了一次，达到了目的。再在本地执行 profile report，发现 test 的时间又减少了3秒，变成了10秒。

现在，我们把优化好的 pipeline 在 ci agent 上执行一下，gradle build 的执行时间来到了 58 秒。加上 cache 操作时间15秒，一共1分13秒。

## 5. 总结

通过以下方式，我们将 ci 上的 gradle build 时间从 5 分钟缩短到了 1 分钟：

1. 通过挂载 gradle cache，避免不必要的依赖下载时间
2. 通过跨 agent 传输文件夹，实现 gradle cache 在不同 agent 之间的传递
3. 通过去除不需要的 gradle task，减少 gradle build 的耗时
4. 通过尽可能地重用 spring test 中的 ApplicationContext，减少 spring boot 在集成测试中的启动次数。


gradle 还提供了其他关于性能的 tips，感兴趣的可以进一步阅读 <https://docs.gradle.org/current/userguide/performance.html>
