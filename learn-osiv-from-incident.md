# 从事故中学习 Spring Open Session In View

## 事故

在一个安静祥和的下午，一条生产环境的告警打乱了所有人的开发节奏：“服务器处理请求的失败率增高至7%”。正要打开监控系统看看情况，又一条告警出现了：“服务响应时间已增加到了每个请求30秒”。赶紧看一眼监控，发现并不是某一个 api 出现问题，而是遍地开花。搜一下错误日志吧，如下：

```
Unable to acquire JDBC Connection; nested exception is org.hibernate.exception.JDBCConnectionException: Unable to acquire JDBC Connection] with root cause java.sql.SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 30000ms.
```

数据库连接池请求不到，等了30秒然后挂掉了。好吧，初步猜测是哪些请求处理太慢了，把数据库连接全部占着，导致其他请求等待数据库连接池超时引发的吧。这时候新的告警又来了：“访问外界系统请求失败率提升”。进去看一眼，发现是我们在调用第三方验证码系统的请求，失败率已逐渐提高至100%，全是502，他们挂了。这个时候我们自己的服务恢复了，请求响应时间回到了正常水平，失败率也逐渐只剩下发验证码的 api 居高不下。看一眼这个调用第三方服务的 api 监控，发现在过去的10分钟里，该 api 响应极慢，后来逐渐崩掉。

## 初步猜测与验证

乍一看事故的原因似乎已经很明晰了，上面的现象也基本符合关于数据库连接池的猜测。在第三方服务完全挂掉之前，响应时间很长的时候，该 api 对数据库连接超时占用，其他线程都需要等待数据库连接，导致响应时间变长甚至失败。等到第三方服务挂掉后，由于我们能直接返回失败，不会继续占用数据库连接，所以其他接口又好了起来。

奇怪的是这种混合数据库和外界系统的 IO 是我们日常开发中极力避免的，为什么还是发生了呢？话不多说快去看一眼代码吧，以下是简化逻辑。

```
// OtpController
@PostMapping("/otps")
public ResponseEntity<RequestOtpResponse> requestOtp(RequestOtpRequest request) {
    String otp = otpService.generateAndSaveOtp(request.getPhoneNumber());
    log.debug("Token saved.");
    otpSender.send(otp);
    return ResponseEntity.ok(RequestOtpResponse.from(otp));
}
// OtpService
@Transactional
public String generateAndSaveOtp(String phoneNumber) {
    String otp = otpGenerator.generateNew();
    return otpRepository.save(new OTP(phoneNumber, otp));
}
// OtpSender
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

```
HikariPool-1 - Pool stats (total=10, active=0, idle=10, waiting=0)
```

Hikari 连接池的默认状态是有10个连接，当前是有10个空闲，然后我们用本地 postman 进行请求，由于我们已经手动延长了该接口的响应时间，postman 一直在等待，从日志我们也马上看到了读写数据库操作完成，

```
Token saved.
```

过了一会连接池的状态更新如下：

```
HikariPool-1 - Pool stats (total=10, active=1, idle=9, waiting=0)
```

我们发现，即使过了很长时间，仍旧有一个连接被占用着。很奇怪但它似乎能够反映生产环境上的情况，让我们多加几个请求，试一下能否产生同样的报错。由于连接池一共10个连接，我们先试上11个请求，如果到目前为止我们的理论正确的话，它们足够产生相应的报错了。30秒过后，果然：

```
java.sql.SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 30002ms.
```

## 找到原因

复现是复现了，可这是为什么呢？理解不了就上网搜一下吧！很快我们发现答案都指向了一个概念：Spring Open Session In View (OSIV)。