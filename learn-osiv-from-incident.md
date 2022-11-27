# 从事故中学习 Spring Open Session In View

## 事故

一天下午，一条生产环境的告警打乱了所有人的开发节奏：“服务器处理请求的失败率增高至7%”。正要打开监控系统看看情况，又一条告警出现了：“服务响应时间已增加到了每个请求30秒”。赶紧看一眼监控，发现并不是某一个 api 出现问题，而是遍地开花。搜一下错误日志吧，如下：

```bash
Unable to acquire JDBC Connection; nested exception is org.hibernate.exception.JDBCConnectionException: Unable to acquire JDBC Connection] with root cause java.sql.SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 30000ms.
```

数据库连接池请求不到，等了30秒然后挂掉了。好吧，初步猜测是哪些请求处理太慢了，把数据库连接全部占着，导致其他请求等待数据库连接池超时引发的吧。这时候新的告警又来了：“访问外界系统请求失败率提升”。进去看一眼，发现是我们在调用第三方验证码系统的请求，失败率已逐渐提高至100%，全是502，他们挂了。神奇的是，这个时候我们自己的服务恢复了，请求响应时间回到了正常水平，失败率也逐渐只剩下发验证码的 api 居高不下。看一眼这个调用第三方服务的 api 监控，发现在过去的10分钟里，该 api 响应极慢，后来逐渐崩掉。

## 初步猜测与验证

乍一看事故的原因似乎已经很明晰了，上面的现象也基本符合关于数据库连接池的猜测。举例来说（示例代码如下），在一个 `@Transactional` （数据库事务）中，先进行了数据库的调用，然后又进行了对外界服务 `name-service` 的调用，然后更新名字。在第三方服务完全挂掉之前，响应时间很长的时候，该 api 对数据库连接超时占用，这样的请求多起来以后，数据库连接池会被占满。其他线程都需要等待数据库连接，导致响应时间变长，或因等待超时而失败。等到第三方服务挂掉后，由于我们能直接拿到失败响应，便不会继续占用数据库连接，所以其他接口又好了起来。

```java
@Transactional
public void changeName(String id) {
    // db io
    String abc = someRepository.findById(id);
    // outgoing http io
    String newName = nameServiceClient.fetchNewName(id);
    abc.setName(newName);
}
```

上面的代码我们一般称为数据库和外界系统的混合 IO，是我们日常开发中极力避免的，可是为什么还是发生了呢？话不多说快去看一眼代码吧，以下是简化逻辑。

```java
// Controller
@PostMapping("/one-time-passwords")
public ResponseEntity<SendOtpResponse> sendOtp(SendOtpRequest request) {
    // 1. save db
    String otp = otpService.createOtp(request.getTarget());
    // 2. outgoing request
    otpSender.send(otp);
    return ResponseEntity.ok(SendOtpResponse.from(otp));
}
// Service
@Transactional
public String createOtp(String target) {
    String code = createNew();
    return otpRepository.save(new OTP(target, code));
}
// SmsSender
public void send(String code) {
    sendHttpRequest(code);
}
```

我们的逻辑其实很简单，生成一个字符串作为 otp，把它存到数据库，然后使用 http 请求通过第三方系统发短信给终端用户。通过查看代码我们发现，数据库 IO 和 http 请求已经被分开了，并没有上面猜测中的问题。

## 本地复现

这个时候我们有点懵了，搞不清楚这个请求能够占用数据库连接的原因。想不到好办法，开始本地复现吧！为了模拟线上超时请求，我们把 `OtpSender` 的 `send` 方法改成强制等待：

```
// OtpSender
public void send(String code) {
    try {
        TimeUnit.SECONDS.sleep(100);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
}
```

既然是 `HikariPool` 的问题，那我们把 `com.zaxxer` 包的 debug 日志打开，然后启动本地服务，看下数据库连接池的占用情况吧。我们发现 `com.zaxxer.hikari.pool.HikariPool` 每隔30秒会打印一次连接池的状态信息：

```bash
HikariPool-1 - Pool stats (total=10, active=0, idle=10, waiting=0)
```

Hikari 连接池的默认状态是有10个连接，当前是有10个空闲，然后我们用本地 postman 进行请求，由于我们已经手动延长了该接口的响应时间，postman 一直在等待，从日志我们也马上看到了读写数据库操作完成，

服务在本地启动了以后，我们先发送一个请求，过了一会连接池的状态更新如下：

```bash
HikariPool-1 - Pool stats (total=10, active=1, idle=9, waiting=0)
```

我们发现，即使过了很长时间，仍有一个连接被占用着。很奇怪但它能够反映生产环境上的情况，让我们多加几个请求，试一下能否产生同样的报错。由于连接池一共10个连接，我们先试上11个请求，如果到目前为止我们的理论正确的话，它们足够产生相应的报错了。另外10个请求发出，30秒过后，果然：

```bash
HikariPool-1 - Pool stats (total=10, active=10, idle=0, waiting=1)
HikariPool-1 - Timeout failure stats (total=10, active=10, idle=0, waiting=0)
HikariPool-1 - Connection is not available, request timed out after 30002ms.
```

如生产环境一样，后面的请求会因等不到数据库连接池而失败，甚至其他接口，只要是需要访问数据库的请求也都会失败，直至超时的访问三方 api 的请求完成，数据库连接得到释放以后，其他接口才能正常访问。

## 找到原因

复现是复现了，可这是为什么呢？理解不了就上网搜一下吧！很快我们发现答案都指向了一个概念：Spring Open Session In View (OSIV)。简单来说，Session per request 是一种事务模式，将存储层的会话与所处理的请求的生命周期绑在一起，从而将业务组件与数据库连接的管理代码解耦。这种模式方便了 hibernate 中 lazy load 的使用。Spring 为此有自己的实现，叫 [OpenSessionInViewInterceptor](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/orm/hibernate5/support/OpenSessionInViewInterceptor.html)，用以提高开发者的生产效率。

这个 `OpenSessionInViewInterceptor` 是默认开启的，它开启时会对服务器处理的请求有以下影响：

1. 当请求到达 spring 时，`OpenSessionInViewInterceptor` 会开启一个 `Hibernate session`，这些 session 并不会直接连接数据库。
2. 本次请求的处理过程中，每次用到存储层会话时，都会使用上面已经创建了的 session。
3. 当请求处理结束时，`OpenSessionInViewInterceptor` 会把之前开启的那个 session 关闭。

它的逻辑是，让框架来处理存储层的会话，让开发者更注重业务相关的实现，而非数据库连接这种看起来比较底层的操作，从而提升开发效率。

了解了 OSIV 以后，我们再回过头来看自己的代码，发现虽然我们没有在数据库 IO 和外界系统 IO 共同存在的方法外加 `@Transactional` 来让它们共处一次事务，但在 service 层的事务结束后，hibernate session 并没有结束，服务与数据库之间的连接也并没有关闭，而是等到了外界请求结束，我们的服务返回响应时，才关闭数据库连接。

为了进一步理解并证实这个逻辑，我们更改本地代码为：

```java
// OtpController
@PostMapping("/one-time-passwords")
public ResponseEntity<RequestOtpResponse> requestOtp(RequestOtpRequest request) throws InterruptedException {
    log.info("sleep 1 start");
    Thread.sleep(30_000);
    log.info("sleep 1 end");
    log.info("connect db start");
    otpService.findAll();
    log.info("connect db end");
    log.info("sleep 2 start");
    Thread.sleep(30_000);
    log.info("sleep 2 end");
    return ResponseEntity.ok(RequestOtpResponse.from("hello"));
}
```

可以看到关键的日志顺序与相关解读如下：

```bash
# 进入请求
Mapped to xxxxxx#requestOtp()
# 启用 OpenEntityManagerInViewInterceptor
Opening JPA EntityManager in OpenEntityManagerInViewInterceptor
sleep 1 start
# 在 sleep 1 期间，虽然有 hibernate session，但是此时并没有过数据库访问，因此也没有占用数据库连接池
HikariPool-1 - Pool stats (total=10, active=0, idle=10, waiting=0)
sleep 1 end
connect db start
# 复用已有 hibernate session
Found thread-bound EntityManager [SessionImpl(473158969<open>)] for JPA transaction
Creating new transaction with name [org.springframework.data.jpa.repository.support.SimpleJpaRepository.findAll]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT,readOnly
# 开启事务
begin
# 查询语句
select xxxxxx from xxxxxx
# 提交事务
committing
Resetting read-only flag of JDBC Connection [HikariProxyConnection@917818296 wrapping conn0: url=jdbc:h2:mem:db user=TEST]
# 没有关闭 EntityManager (hibernate session)
Not closing pre-bound JPA EntityManager after transaction
connect db end
sleep 2 start
# 在访问过数据库之后，虽然事务已经提交，但是 hibernate session 并没有关闭，且仍然占用连接池
HikariPool-1 - Pool stats (total=10, active=1, idle=9, waiting=0)
sleep 2 end
# 在响应返回之前才关掉 EntityManager (hibernate session)
Closing JPA EntityManager in OpenEntityManagerInViewInterceptor
Completed 200 OK
```

看起来 OSIV 的确是这次事故的根源了。

## 问题解决

既然道理明白了，我们就关闭 OSIV 来看下问题是否解决。在 `application.yml` 里面加上 `spring.jpa.open-in-view=false`，然后重新启动服务，安排上20个并发请求，发现此时并不会造成数据库连接的长期占用，Hikari 连接池也始终在用的时候才会 active，在测试中所有的连接池日志中 active 都是 0，问题得到了解决。

需要注意的是，在禁用 OSIV 以后，它所带来的便利也没有了，比如一些默认为 lazy fetch 的 ToMany 关系可能会抛出 `LazyInitializationException`。由于项目中有较多的集成测试，只要测试全过，便可以保证现有代码中没有 lazy fetch 的情况。再加上大家一起看了实体间的关系，更加确认了此项更改的安全性。

将更改部署到 staging 环境，并做全面的端到端回归测试以后，我们将更改部署到了生产环境，一切又恢复了平静。

## 其他

如前面所说，Session per request 可以让开发者更关注业务，减少在管理数据库连接上浪费时间，从而提升开发效率。多数情况确实是这样的，但也正是因为一些工具的过于便捷，导致一些本该由开发者积极思考的问题被忽略。正如这次的数据库连接池问题，或许默认关闭 OSIV，可以更好地让开发者去主动思考连接的建立与释放呢。
