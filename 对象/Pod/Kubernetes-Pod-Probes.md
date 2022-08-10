#  kuberenetes pod Liveness, Readiness and Startup Probes
tags: Pod,探针,健康检测
<!-- catalog:~Probes~ -->


[
![在这里插入图片描述](https://img-blog.csdnimg.cn/3a330d60c83345aba9bb379affd2a492.png)](https://www.rottentomatoes.com/m/ex_machina)




## 1. Pod 状态
Pod的status字段是 PodStatus 对象，其中包含一个phase字段。
| 值         | 描述                                                                                |
|-----------|-----------------------------------------------------------------------------------|
| Pending   | 该Pod已被Kubernetes系统接受，但是尚未创建一个或多个Container映像。这包括计划之前的时间以及通过网络下载图像所花费的时间，这可能需要一段时间。 |
| Running   | Pod已绑定到节点，并且所有容器都已创建。至少一个容器仍在运行，或者正在启动或重新启动。                                      |
| Succeeded | Pod中的所有容器已成功终止，并且不会重新启动。                                                          |
| Failed    | Pod中的所有容器均已终止，并且至少一个容器因故障而终止。也就是说，容器要么以非零状态退出，要么被系统终止。                            |
| Unknown   | 由于某些原因，通常由于与Pod主机通信时出错而无法获得Pod的状态。                                                |

## 2. Pod 生命周期条件
Pod具有PodStatus，该状态具有PodConditions数组 ，该Pod已通过或未通过。PodCondition数组的每个元素都有六个可能的字段：

 - 该`lastProbeTime`字段提供上次探测Pod条件的时间戳。
 - 该`lastTransitionTime`字段提供Pod上一次从一种状态转换为另一种状态的时间戳。
 - 该`message`字段是人类可读的消息，指示有关过渡的详细信息。
 - 该`reason`字段是条件最后一次转换的唯一单词CamelCase原因。
 - 该`status`字段是一个字符串，可能的值为“ True”，“ False”和“ Unknown”。
 - 该`type`字段是具有以下可能值的字符串：

```bash
PodScheduled：Pod已调度到一个节点；
Ready：Pod能够处理请求，应将其添加到所有匹配服务的负载平衡池中；
Initialized：所有初始化容器 已成功启动；
Unschedulable：例如，由于缺乏资源或其他限制，调度程序无法立即调度Pod；
ContainersReady：Pod中的所有容器已准备就绪。
```
## 3. 容器探针
有三种类型的处理程序：

 - `ExecAction`：在Container中执行指定的命令。如果命令以状态代码0退出，则认为诊断成功。
 - `TCPSocketAction`：对指定端口上的容器的IP地址执行TCP检查。如果端口打开，则认为诊断成功。
 - `HTTPGetAction`：对指定端口和路径上的容器的IP地址执行HTTP
   Get请求。如果响应的状态码大于或等于200且小于400，则认为诊断成功。

每个探针具有以下三个结果之一：

 - 成功：容器通过了诊断。
 - 失败：容器无法通过诊断。
 - 未知：诊断失败，因此不应采取任何措施。

kubelet可以选择对正在运行的Container进行两种探测并对其做出反应：

 - `livenessProbe`：指示容器是否正在运行。如果活动探针失败，则kubelet将杀死Container，并且Container将接受其重新启动策略。如果容器未提供活动性探针，则默认状态为Success。
 - `readinessProbe`：指示容器是否准备好服务请求。如果就绪探针失败，则端点控制器将从与Pod匹配的所有服务的端点中删除Pod的IP地址。初始延迟之前的默认就绪状态为Failure。如果容器未提供就绪探测器，则默认状态为Success。
## 4. 什么时候应该使用活动或就绪探针？
如果您的Container中的进程在遇到问题或变得不正常时能够自行崩溃，则不一定需要进行活动调查；kubelet将根据Pod的自动执行正确的操作`restartPolicy`。

如果您希望在探测失败时杀死并重启容器，请指定活动探测，并指定restartPolicyAlways或OnFailure。

如果您仅想在探测成功后才开始向Pod发送流量，请指定就绪探测器。在这种情况下，就绪探针可能与活动探针相同，但是规范中存在就绪探针意味着Pod将在不接收任何流量的情况下启动，并且仅在探针开始成功之后才开始接收流量。如果您的容器需要在启动过程中加载大型数据，配置文件或迁移，请指定准备情况探针。

如果希望您的Container能够自行进行维护，则可以指定一个就绪探针，以检查特定于与活跃探针不同的就绪端点。

> 请注意，如果您只想在删除Pod时便能够清空请求，则不一定需要准备就绪探测器；删除后，无论是否准备就绪探针，Pod都会自动将自己置于未就绪状态。等待Pod中的Container停止时，Pod仍处于未就绪状态。

## 5. 配置技巧
### 5.1 使用命名端口

```bash
ports:
- name: liveness-port
  containerPort: 8080
  hostPort: 8080

livenessProbe:
  httpGet:
    path: /healthz
    port: liveness-port
```

### 5.2 使用启动探测器保护慢启动容器

```bash
ports:
- name: liveness-port
  containerPort: 8080
  hostPort: 8080

livenessProbe:
  httpGet:
    path: /healthz
    port: liveness-port
  failureThreshold: 1
  periodSeconds: 10

startupProbe:
  httpGet:
    path: /healthz
    port: liveness-port
  failureThreshold: 30
  periodSeconds: 10
```
幸亏有启动探测，应用程序将会有最多 5 分钟(30 * 10 = 300s) 的时间来完成它的启动。 一旦启动探测成功一次，存活探测任务就会接管对容器的探测，对容器死锁可以快速响应。 如果启动探测一直没有成功，容器会在 300 秒后被杀死，并且根据 restartPolicy 来设置 Pod 状态。

![在这里插入图片描述](https://img-blog.csdnimg.cn/7465ae7606a3468fb3c8c18e4ebfe5e6.gif#pic_center)

## 6. 探测方法
### 6.1 http 探测
```bash
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - args:
    - /server
    image: k8s.gcr.io/liveness
    livenessProbe:
      httpGet:
        # when "host" is not defined, "PodIP" will be used
        # host: my-host
        # when "scheme" is not defined, "HTTP" scheme will be used. Only "HTTP" and "HTTPS" are allowed
        # scheme: HTTPS
        path: /healthz
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: Awesome
      initialDelaySeconds: 15
      periodSeconds: 3
      timeoutSeconds: 1
    name: liveness
```
`periodSeconds` 字段指定了 kubelet 每隔 3 秒执行一次存活探测。 `initialDelaySeconds` 字段告诉 kubelet 在执行第一次探测前应该等待 3 秒。 kubelet 会向容器内运行的服务（服务会监听 8080 端口）发送一个 HTTP GET 请求来执行探测。 如果服务器上 /healthz 路径下的处理程序返回成功代码，则 kubelet 认为容器是健康存活的。 如果处理程序返回失败代码，则 kubelet 会杀死这个容器并且重新启动它。

任何大于或等于 200 并且小于 400 的返回代码标示成功，其它返回代码都标示失败。

### 6.2 cmd 探测

```bash
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```
 `periodSeconds` 字段指定了 kubelet 应该每 5 秒执行一次存活探测。 `initialDelaySeconds` 字段告诉 kubelet 在执行第一次探测前应该等待 5 秒。 kubelet 在容器内执行命令 cat `/tmp/healthy` 来进行探测。 如果命令执行成功并且返回值为 0，kubelet 就会认为这个容器是健康存活的。 如果这个命令返回非 0 值，kubelet 会杀死这个容器并重新启动它。
###  6.3 TCP 探测

```bash
apiVersion: v1
kind: Pod
metadata:
  name: goproxy
  labels:
    app: goproxy
spec:
  containers:
  - name: goproxy
    image: k8s.gcr.io/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```
下面这个例子同时使用就绪和存活探测器。kubelet 会在容器启动 5 秒后发送第一个就绪探测。 这会尝试连接 goproxy 容器的 8080 端口。 如果探测成功，这个 Pod 会被标记为就绪状态，kubelet 将继续每隔 10 秒运行一次检测。

除了就绪探测，这个配置包括了一个存活探测。 kubelet 会在容器启动 15 秒后进行第一次存活探测。 就像就绪探测一样，会尝试连接 goproxy 容器的 8080 端口。 如果存活探测失败，这个容器会被重新启动。


参考：

 - [配置存活、就绪和启动探测器](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
 - [Kubernetes Startup Probes - Examples & Common Pitfalls](https://loft.sh/blog/kubernetes-startup-probes-examples-common-pitfalls/)




