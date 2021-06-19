#Kubernetes环境下保证服务高可用的部署实践

>阅读本文需要读者了解Kubernetes(k8s)的基础概念，如pod、deployment、service。本文讨论的更多的是如何利用k8s本身提供的服务，低成本地达到我们的应用服务的可用性需求。
>由于篇幅过长的话影响阅读体验，本文对各种配置都采用官网链接的方式，不在文中贴代码了。

近几年，在我们的新项目上，Kubernetes作为部署环境使用颇为广泛。在微服务比较流行的背景下，创建新的服务也是经常发生。服务的高可用的必要性自然不用多说，虽然实现高可用的程度和所需付出的努力也是正相关的，但并不线性。经过不断的摸索，我总结了我接触的近几个服务中，逐渐稳定下来的性价比高的常用高可用配置。

##1.保证自身运行与更新的平稳

###1.1livenessProbe

很多应用在长时间运行以后，都可能进入一些不健康的状态，比如处理速度变慢，内存压力变大，甚至死锁，业务服务中止等。这时候虽然容器还在继续运行，但已经没有了处理业务的能力，且通常重启pod便可以解决问题。为了自动辨识并修复这种情况，k8s提供了一种探针[livenessProbe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-command)，顾名思义，它可以让k8s知道这个容器是否还“活着”，如果容器已经停止服务，则kubelet会重启容器。`livenessProbe`支持多种配置，如探针形式、频率，和失败次数等。这样我们就在一定程度上缓解了业务服务“名存实亡”的问题。

###1.2readinessProbe

然而，有些情况业务服务不工作时，重启并不能解决问题。比如我们起一个spring服务的pod，它启动起来就是需要那么几十秒，通常为了等待服务的启动，我们会将`livenessProbe`延后，以免错杀正在启动的容器。但是在业务进程启动的过程中，k8s感知到的是容器正在运行，因此当你配置了[service](https://kubernetes.io/docs/concepts/services-networking/service/)的时候，service会把流量转发到还没有就绪的容器中，造成请求失败。

为了应对这种情况，k8s提供了另一种探针[readinessProbe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-readiness-probes)，k8sservice会根据`readinessProbe`返回成功与否的情况，决定是否将流量转发到相应的容器中。这样一来，在服务有多副本的情况下，只要我们还有正常运行的服务，对于外部使用者来说，他的请求总能得到应有的回应。同`livenessProbe`一样，`readinessProbe`也支持探针形式、频率，和失败次数等多种配置。

###1.3部署版本检查

持续集成/持续部署作为一种最佳实践如今已被广为采用，它的落地依赖于一个好的流水线设计，我们之所以能做到持续部署，是因为我们的流水线有足够多的检查和测试，且能够在异常情况下挂掉，从而中止问题版本部署到生产环境。由于k8s对组件的管理是[声明式](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)的，想要更新部署版本只需要在k8s中更新一个deployment配置，我们更新配置的指令也会在k8s接受配置后直接返回，并不会等到新的服务启动。那么问题就出现了，如果新版本的pod启动不起来，我们如何知道？如果新启动的服务迟迟没有Ready，我们又如何知道？

我们的方法比较淳朴，在服务中暴露一个状态接口，返回一些用于检测的环境变量，比如流水线buildnumber，commit号等，然后在流水线中循环使用curl请求，直到服务能够返回了期望的版本，才会通过，否则会报错，让流水线挂掉。如果通过的话说明至少一个新启动的pod已经Ready，其他正在rollingupdate的副本也基本不会有什么问题，我们就认为部署成功，可以继续进行下一步，比如自动化测试或者部署到下一个环境。

目前我还能想到的一种方法是利用k8s的新特性[kubectlwait](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#wait)。这个命令会在等待条件未满足之前一直hang住，直到timeout，用在流水线里看起来更加优雅。我们可以在部署的时候给pod打上标签，比如`app:my-app`,`commit:abcdefg`，然后通过`kubectlwait--for=condition=Readypod-lapp=myapp-lcommit=abcdefg--timeout=120s`，来保证我们想要的版本已经可以提供外界服务。

>项目初期我们通过以上三种途径，基本保证了服务在持续集成中的零宕机部署，然而上面提到方式比较简单，适用于主干开发、持续集成/持续部署，靠的是各微服务滚动更新，成本比蓝绿部署低一些。如果对可用性要求更高，可以考虑蓝绿部署等更复杂的方式。


##2.抵御内外干扰

###2.1HorizontalPodAutoscaler

如今大部分系统面临的流量都或多或少的有一些潮汐特性，比如吃饭时间点外卖的人更多些，此时由于流量过大，相关服务同时处理的请求过多，很容易引发各种问题，最终导致服务不可用。然而依照这个流量部署外卖服务器，等过了饭点，又会有大量的服务器处于空闲状态。快递、打车、支付，很多行业都是这样，课件服务的自动伸缩成了很多企业标配的需求。幸运的是，在k8s生态下，它也不是什么难事。

[HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)是k8s为我们提供的开箱即用的功能，我们只需要指定该服务可接受的最大(`maxReplicas`)与最小副本数(`minReplicas`)，目标cpu和内存使用率，则k8s会自动为我们进行调整，当服务负载过高时，自动增加副本数，反之则自动减少副本数，以维持各容器的实际负载与目标负载相匹配。这样一来，我们既可以抵御高峰流量，又可以在流量低谷时保持一定的经济性。

<!--HorizontalPodAutoscaler官网样例：
```yaml
apiVersion:autoscaling/v1
kind:HorizontalPodAutoscaler
metadata:
name:php-apache
spec:
scaleTargetRef:
apiVersion:apps/v1
kind:Deployment
name:php-apache
minReplicas:1
maxReplicas:10
targetCPUUtilizationPercentage:50
```-->

###2.2podAntiAffinity

有时影响我们服务稳定性的，不仅是外部因素，还可能是内部需求。比如当k8s集群为开发团队提供便利性的同时，还需要运维同事对其进行维护。作为k8snode的虚拟机也好，其背后的物理机也好，都需要一些定期重启的机制，来保证其稳定性。本来我以为这和我们开发团队没关系的，直到有一天，我们的“零宕机”的服务出现了几分钟的宕机。

简单沟通后我们得到的原因是，运维团队有一个定时任务，每隔一段时间便会清理运行时间大于n小时的节点。在清理node（通过`drainnode`）的时候，k8s会做deletepod的动作，将该节点上所有的pod删掉，同时调度器也不会调度新的pod到该node上，从而最终移除该node。至于被删掉的pod，由于deployment中定义了它应有的副本数，当k8s发现实际pod数量少于声明数量时，就会在其他node上重新创建pod。当天比较巧的是，我们同一个服务的全部2个副本都在需要删除的node上运行，所以该服务的2个副本都被删除了，而就在老的pod被删除之后，新的podready之前，我们宕机了。

在搞明白宕机的原因以后，当时我们得到的建议是使用[podAntiAffinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)。通过定义`podAntiAffinity`，可以告知k8s调度器在创建pod时，避开相似的pod，也就是说，在这种情况下，当一个node上已经运行着一个符合条件的pod时，新的pod可以选择不被调度到这个node上，根据使用`requiredDuringSchedulingIgnoredDuringExecution`还是`preferredDuringSchedulingIgnoredDuringExecution`，决定了它是强制条件还是preferred条件。如果使用强制条件，那么当没有符合条件的node可用时，新的pod会一直等待。我们通过`requiredDuringSchedulingIgnoredDuringExecution`保证了同一服务的不同副本不会被调用到同一节点上，从而避免了删除一个node导致整个服务宕机的情况。

>当然，是否配置上述强制条件必须结合实际情况，比如自动扩缩容的数量如果大于node数量，则强制条件会让多出来的pod始终处于等待状态。

<!--podAntiAffinity官网样例截取：
```yaml
apiVersion:apps/v1
kind:Deployment
metadata:
name:redis-cache
spec:
selector:
matchLabels:
app:store
replicas:3
template:
metadata:
labels:
app:store
spec:
affinity:
podAntiAffinity:
requiredDuringSchedulingIgnoredDuringExecution:
-labelSelector:
matchExpressions:
-key:app
operator:In
values:
-store
topologyKey:"kubernetes.io/hostname"
containers:
-name:redis-server
image:redis:3.2-alpine
```-->

###2.3PodDisruptionBudget

对于定期清理node造成的服务宕机，上述配置`podAntiAffinity`的方法显然是不够的，如果一次需要删除3个node，而我们的服务恰巧在其中的两个，也还是要遭殃。好在k8s还提供了[PodDisruptionBudget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)配置，这名字非常表义，他可以让我们声明一个最小可用数`minAvailable`，或最大不可用数`maxUnavailable`，当执行`drainnode`的时候，待删除的pod先会等待，同时在其他node上创建新的pod，保证删除后副本数仍然达标，然后才再删掉需要删除的pod。这样就从根本上解决了问题。
<!--
PodDisruptionBudget官网样例：
```yaml
apiVersion:policy/v1
kind:PodDisruptionBudget
metadata:
name:zk-pdb
spec:
minAvailable:2
selector:
matchLabels:
app:zookeeper
```-->

###2.4Gracefulshutdown

上述这些配置似乎已经能满足大多数情况了，这里再补充一点，就是优雅退出，推荐一篇博客<https://learnk8s.io/graceful-shutdown>。简而言之，就是k8s在删除pod的时候，容器会先进入`Terminating`状态，接着kubelet会先给容器发一个`SIGTERM`，然后等容器退出，当容器在一定时间内没能自己退出时，kubelet会给容器发送一个`SIGKILL`，强制容器退出。

也就是说，即便我们做到了上面的那些，保证了只有达到条件时，才会删除pod，也不能保证老的pod中正在进行的任务顺利完成，因为它可能被中途`SIGKILL`掉，对于用户来说，还是中途宕机。好在这段从Terminating状态到`SIGKILL`信号发出的时间是可配置的，即[terminationGracePeriodSeconds](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)。这里我们可以通过一些策略，设置一个合理的`terminationGracePeriodSeconds`，对我们的服务进行保障。

>我们使用的springboot也支持[gracefulshutdown](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.graceful-shutdown)，但默认不是开启的，需要在`application.yml`中指定`server.shutdown=graceful`,同时也可以设置gracefulshutdown的时间`spring.lifecycle.timeout-per-shutdown-phase=20s`。

此外，在pod的生命周期中，我们还可以定义[preStop](https://kubernetes.io/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/)指令，在发送`SIGTERM`之前做一些必要的等待、检查等工作，用以保证业务在pod退出之前被妥善处理。

<!--```yaml
apiVersion:v1
kind:Pod
metadata:
name:lifecycle-demo
spec:
containers:
-name:lifecycle-demo-container
image:nginx
lifecycle:
postStart:
exec:
command:["/bin/sh","-c","echoHellofromthepostStarthandler>/usr/share/message"]
preStop:
exec:
command:["/bin/sh","-c","nginx-squit;whilekillall-0nginx;dosleep1;done"]
```-->

>更多阅读：[GracefulNodeShutdownGoesBeta](https://kubernetes.io/blog/2021/04/21/graceful-node-shutdown-beta/)

##结语

这些实践已经基本在我们项目作为模板，每当我们创建新项目的时候都会默认应用，基本上都是开箱即用的东西，我们加几行配置就可以使用，如今服务稳定性也还不错。欢迎大家一起讨论，拍砖！

