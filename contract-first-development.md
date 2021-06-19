# 契约先行开发


## 契约维护问题

如今微服务凭借其灵活、易开发、易扩展等优势深入人心，不同服务之间的集成和交互日渐繁多且复杂。这些服务之间交互的方式是多样的，常见的有 HTTP 请求和消息队列，在它们交互的过程中，会有服务的版本演进，交互信息的格式或方式就会产生变化，前后版本的接口可能并不兼容，然而不同服务的开发进度有快有慢，各团队的优先级有高有低，服务间交互方式的匹配性就成了一个问题。这里，不同团队之间，对服务间如何进行发送和接受消息所能达成的共同理解，我们称之为契约 (contract)。如何采用一个合理的机制，维护服务间契约，使服务提供方和消费房能够在不造成事故的前提下，保持各自的高效开发，越来越成为各团队日常开发中要面对的问题。

## 契约测试

[契约测试](https://docs.pact.io/#what-is-contract-testing) (contract testing) 就是在这样的背景下应运而生，以下引用 pact 官网的定义：

> Contract testing is a technique for testing an integration point by checking each application in isolation to ensure the messages it sends or receives conform to a shared understanding that is documented in a "contract".

*契约测试是一种测试集成点的技术，它通过隔离检查每个应用程序，以确保其发送或接收的消息，符合记录在“契约”中的共同理解。*

也就是说，在测试己方服务时，通过使用测试替身 (test double)，让它能够模仿我们所依赖的外界系统，返回相对真实的消息响应，从而让我方团队在尽可能保证与外界系统兼容的前提下，避免受到外界系统宕机或开发新版本等影响，提升开发效率。再结合消费者驱动开发的优势，避免服务提供端浪费精力去实现不必要的功能，很多团队采用了消费者驱动的契约测试 (consumer-driven contract test) 的实践，。

![test double](./pictures/consumer-driven-contract-test.png)

在契约测试的帮助下，很多团队真正提升了开发效率，掌握了自己的节奏，但也有些团队发现效果并不明显，因为契约测试带来的收益并不是免费的。


契约测试麻烦在哪里：
契约测试模式下，团队的沟通闭环
尝试改变沟通闭环：沟通提前
契约先行开发
生成代码
契约维护与更新
通过契约和代码生成避免无效请求和无效响应

