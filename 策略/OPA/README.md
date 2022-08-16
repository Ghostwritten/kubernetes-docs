#  Open Policy Agent(OPA) 
tags: OPA,策略

## 1. OPA 介绍
开放策略代理（OPA，发音为“ oh-pa”）是一个开放源代码的通用策略引擎，它统一了整个堆栈中的策略执行。OPA提供了一种高级的声明性语言，使您可以将策略指定为代码和简单的API，以减轻软件决策的负担。您可以使用OPA在微服务，Kubernetes，CI / CD管道，API网关等中实施策略。

`OPA` 最初是由 `Styra` 公司在 2016 年创建并开源的项目，目前该公司的主要产品就是提供可视化策略控制及策略执行的可视化 `Dashboard` 服务的。

OPA 首次进入 `CNCF` 并成为 `sandbox` 级别的项目是在 2018 年， 在 2021 年的 2 月份便已经从 `CNCF` 毕业，这个过程相对来说还是比较快的，由此也可以看出 `OPA` 是一个比较活跃且应用广泛的项目。


**透过现象看本质，策略就是一组规则，请求发送到引擎，引擎根据规则来进行决策**。OPA 并不负责具体任务的执行，它仅负责决策，需要决策的请求通过 JSON 的方式传递给 OPA ，在 OPA 决策后，也会将结果以 JSON 的形式返回。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021051700011275.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpeGloYWhhbGVsZWhlaGU=,size_16,color_FFFFFF,t_70)

##  2. OPA 解决了哪些问题
OPA通过评估查询输入以及针对策略和数据来生成策略决策。OPA和Rego是域无关的，因此您可以描述策略中几乎所有类型的不变式。例如：

 1. 哪些用户可以访问哪些资源；
 2. 允许哪些子网出口流量；
 3. 必须将工作负载部署到哪个群集；
 4. 可以从哪些注册表二进制文件下载；
 5. 容器可以执行哪些OS功能；
 6. 可以在一天的哪个时间访问系统；
 7. 需要策略控制用户是否可登陆服务器或者做一些操作；
 8. 需要策略控制哪些项目/哪些组件可进行部署；
 9. 需要策略控制如何访问数据库；
 10.需要策略控制哪些资源可部署到 Kubernetes 中； 

策略决策不仅限于简单的是/否或允许/拒绝答案。像查询输入一样，您的策略可以生成任意结构化数据作为输出。
![在这里插入图片描述](https://img-blog.csdnimg.cn/bb67bc706e524598ad72c7d53de3519c.png?)
但是对于这些场景或者软件来说，配置它们的策略是需要与该软件进行耦合的，彼此是不统一，不通用的。管理起来也会比较混乱，带来了不小的维护成本。

OPA 的出现可以将各处配置的策略进行统一，极大的**降低了维护成本**。以及将策略与对应的软件/服务进行解耦，方便进行移植/复用。
![在这里插入图片描述](https://img-blog.csdnimg.cn/bb6e1d6149d3406f92ea406d150d0a55.png?)

假设您在具有以下系统的组织中工作：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210517000934441.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hpeGloYWhhbGVsZWhlaGU=,size_16,color_FFFFFF,t_70)
系统中包含三种组件：

 - 服务器暴露零层或多个协议（例如，http，ssh等等）
 - 网络连接服务器，可以是公共的或私有的。公共网络已连接到Internet。
 - 端口将服务器连接到网络。

所有服务器，网络和端口均由脚本设置。该脚本接收系统的JSON表示作为输入：


```c
{
    "servers": [
        {"id": "app", "protocols": ["https", "ssh"], "ports": ["p1", "p2", "p3"]},
        {"id": "db", "protocols": ["mysql"], "ports": ["p3"]},
        {"id": "cache", "protocols": ["memcache"], "ports": ["p3"]},
        {"id": "ci", "protocols": ["http"], "ports": ["p1", "p2"]},
        {"id": "busybox", "protocols": ["telnet"], "ports": ["p1"]}
    ],
    "networks": [
        {"id": "net1", "public": false},
        {"id": "net2", "public": false},
        {"id": "net3", "public": true},
        {"id": "net4", "public": true}
    ],
    "ports": [
        {"id": "p1", "network": "net1"},
        {"id": "p2", "network": "net3"},
        {"id": "p3", "network": "net2"}
    ]
}
```
当天早些时候，您的老板告诉您必须实施的新安全策略：

 - Servers reachable from the Internet must not expose the insecure  'http' protocol.
从Internet可访问的服务器不能暴露不安全的“http”协议。
 - Servers are not allowed to expose the 'telnet' protocol.
服务器不允许公开'telnet'协议

配置了服务器，网络和端口并且合规团队希望定期审核系统以查找违反策略的服务器时，必须执行该策略。

您的老板已要求您确定OPA是否适合实施该政策。


##  3. rego介绍
`OPA` 中的策略是以 `Rego` 这种 `DSL`(Domain Specific Language) 来表示的。

`Rego` 受 `Datalog`（https://en.wikipedia.org/wiki/Datalog） 的启发，并且扩展了 Datalog 对于结构化文档模型的支持，方便以 JSON 的方式对数据进行处理。

`Rego` 允许策略制定者可以专注于返回内容的查询而不是如何执行查询。同时 OPA 中也内置了执行规则时的优化，用户可以默认使用。

`Rego` 允许我们使用规则（if-then）封装和重用逻辑，规则可以是完整的或者是部分的。

每个规则都是由头部和主体组成。在 Rego 中，如果规则主体对于某些变量赋值为真，那么我们说规则头为真。可以通过绝对路径引用任何加载到 OPA 中的规则来查询它的值。规则的路径总是：data..（规则生成的所有值都可以通过全局 data 变量进行查询。例如，下方示例中的 `data.example.rules.any_public_networks`

完整规则是将单个值分配给变量的 `if-then` 语句。
![在这里插入图片描述](https://img-blog.csdnimg.cn/19a6a370896d449caffc2ce320f1c058.png?)
部分规则是生成一组值并将该组分配给变量的 `if-then` 语句
![在这里插入图片描述](https://img-blog.csdnimg.cn/49233207e9d0437ab8c4e16c85dc7a37.png?)
逻辑或是要在 `Rego` 中定义多个具有相同名称的规则。（查询中将多个表达式连接在一起时，表示的是逻辑 AND）
![在这里插入图片描述](https://img-blog.csdnimg.cn/2edfa8c07a724726b01f1ba307e88b0f.png?)

## 4. OPA 安装
本节说明如何直接查询OPA并在自己的计算机上与其交互。

1.下载OPA
要开始从GitHub版本下载适用于您平台的OPA二进制文件，请执行以下操作：

在macOS（64位）上：

```bash
curl -L -o opa https://openpolicyagent.org/downloads/v0.28.0/opa_darwin_amd64
```

在Linux（64位）上：

```bash
curl -L -o opa https://openpolicyagent.org/downloads/v0.28.0/opa_linux_amd64
chmod 755 ./opa
cp opa /usr/local/bin/
$ opa version
Version: 0.35.0
Build Commit: a54537a
Build Timestamp: 2021-12-01T02:11:47Z
Build Hostname: 9e4cf671a460
Go Version: go1.17.3
WebAssembly: unavailable
```
容器

```bash
$ docker run --rm  openpolicyagent/opa:0.35.0 version    
Version: 0.35.0
Build Commit: a54537a
Build Timestamp: 2021-12-01T02:10:31Z
Build Hostname: 4ee9b086e5de
Go Version: go1.17.3
WebAssembly: available
```

## 5. OPA 运行 

与OPA交互的最简单方法是通过命令行使用opa eval子命令。`opa eval`是一把瑞士军刀，可用于评估任意的Rego表达式和策略。opa eval支持大量用于控制评估的选项。常用标志包括：

旗帜	短的	描述

 - --bundle	-b	将捆绑包文件或目录加载到OPA中。该标志可以重复。
 - --data	-d	将策略或数据文件加载到OPA中。该标志可以重复。
 - --input	-i	加载数据文件并将其用作input。该标志不能重复。
 - --format	-f	设置要使用的输出格式。默认值为json，并且旨在用于程序设计。该pretty格式发出了更多人类可读的输出。
 - --fail	不适用	如果查询未定义，则以非零的退出代码退出。
 - --fail-defined	不适用	如果查询不是未定义的，则以非零的退出代码退出。

例如：

input.json：

```bash
{
    "servers": [
        {"id": "app", "protocols": ["https", "ssh"], "ports": ["p1", "p2", "p3"]},
        {"id": "db", "protocols": ["mysql"], "ports": ["p3"]},
        {"id": "cache", "protocols": ["memcache"], "ports": ["p3"]},
        {"id": "ci", "protocols": ["http"], "ports": ["p1", "p2"]},
        {"id": "busybox", "protocols": ["telnet"], "ports": ["p1"]}
    ],
    "networks": [
        {"id": "net1", "public": false},
        {"id": "net2", "public": false},
        {"id": "net3", "public": true},
        {"id": "net4", "public": true}
    ],
    "ports": [
        {"id": "p1", "network": "net1"},
        {"id": "p2", "network": "net3"},
        {"id": "p3", "network": "net2"}
    ]
}
```
example.rego:

```bash
package example

default allow = false                               # unless otherwise defined, allow is false

allow = true {                                      # allow is true if...
    count(violation) == 0                           # there are zero violations.
}

violation[server.id] {                              # a server is in the violation set if...
    some server
    public_server[server]                           # it exists in the 'public_server' set and...
    server.protocols[_] == "http"                   # it contains the insecure "http" protocol.
}

violation[server.id] {                              # a server is in the violation set if...
    server := input.servers[_]                      # it exists in the input.servers collection and...
    server.protocols[_] == "telnet"                 # it contains the "telnet" protocol.
}

public_server[server] {                             # a server exists in the public_server set if...
    some i, j
    server := input.servers[_]                      # it exists in the input.servers collection and...
    server.ports[_] == input.ports[i].id            # it references a port in the input.ports collection and...
    input.ports[i].network == input.networks[j].id  # the port references a network in the input.networks collection and...
    input.networks[j].public                        # the network is public.
}
```

执行：

```c
root@master:~/cks/opa# ./opa eval "1*2+3"
{
  "result": [
    {
      "expressions": [
        {
          "value": 5,
          "text": "1*2+3",
          "location": {
            "row": 1,
            "col": 1
          }
        }
      ]
    }
  ]
}
```

```c
root@master:~/cks/opa# ./opa eval -i input.json -d example.rego "data.example.violation[x]"
{
  "result": [
    {
      "expressions": [
        {
          "value": "ci",
          "text": "data.example.violation[x]",
          "location": {
            "row": 1,
            "col": 1
          }
        }
      ],
      "bindings": {
        "x": "ci"
      }
    },
    {
      "expressions": [
        {
          "value": "busybox",
          "text": "data.example.violation[x]",
          "location": {
            "row": 1,
            "col": 1
          }
        }
      ],
      "bindings": {
        "x": "busybox"
      }
    }
  ]
}
```

```bash
root@master:~/cks/opa# ./opa eval --fail-defined -i input.json -d example.rego "data.example.violation[x]"
{
  "result": [
    {
      "expressions": [
        {
          "value": "ci",
          "text": "data.example.violation[x]",
          "location": {
            "row": 1,
            "col": 1
          }
        }
      ],
      "bindings": {
        "x": "ci"
      }
    },
    {
      "expressions": [
        {
          "value": "busybox",
          "text": "data.example.violation[x]",
          "location": {
            "row": 1,
            "col": 1
          }
        }
      ],
      "bindings": {
        "x": "busybox"
      }
    }
  ]
}
root@master:~/cks/opa# echo $?
1
```
## 6. OPA run（互动式）
OPA包括一个交互式外壳程序或REPL（读取-评估-打印循环）。您可以使用REPL来试验策略并为新策略创建原型。

要启动REPL，只需：

```bash
./opa run
```

当您在REPL中输入语句时，OPA会对它们进行评估并打印结果。

```c
> true
true
> 3.14
3.14
> ["hello", "world"]
[
  "hello",
  "world"
]
```
大多数REPL允许您定义以后可以引用的变量。OPA允许您执行类似的操作。例如，您可以定义一个pi常量，如下所示：

```bash
> pi := 3.14
```
定义“ pi”后，您将查询该值并根据该值编写表达式：

```bash
> pi
3.14
> pi > 3
true
```
通过按`Control-D`或键入`exit`以下命令退出REPL ：

```bash
> exit
```
您可以通过在命令行上传递策略和数据文件来将它们加载到REPL中。默认情况下，JSON和YAML文件植于下data。

```bash
opa run input.json
```
运行一些查询以查找数据：

```bash
> data.server[0].protocols[1]
undefined
> data.servers[0].protocols[1]
"ssh"
> data.servers[i].protocols[j]
+---+---+------------------------------+
| i | j | data.servers[i].protocols[j] |
+---+---+------------------------------+
| 0 | 0 | "https"                      |
| 0 | 1 | "ssh"                        |
| 1 | 0 | "mysql"                      |
| 2 | 0 | "memcache"                   |
| 3 | 0 | "http"                       |
| 4 | 0 | "telnet"                     |
+---+---+------------------------------+
> net := data.networks[_]; net.public
+-----------------------------+
|             net             |
+-----------------------------+
| {"id":"net3","public":true} |
| {"id":"net4","public":true} |
+-----------------------------+

```
要将数据文件设置为inputREPL中的文档，请在文件路径前添加前缀：

```bash
opa run example.rego repl.input:input.json
```

```bash
> data.example.public_server[s]
+-------------------------------------------------------------------+-------------------------------------------------------------------+
|                                 s                                 |                   data.example.public_server[s]                   |
+-------------------------------------------------------------------+-------------------------------------------------------------------+
| {"id":"app","ports":["p1","p2","p3"],"protocols":["https","ssh"]} | {"id":"app","ports":["p1","p2","p3"],"protocols":["https","ssh"]} |
| {"id":"ci","ports":["p1","p2"],"protocols":["http"]}              | {"id":"ci","ports":["p1","p2"],"protocols":["http"]}              |
+-------------------------------------------------------------------+-------------------------------------------------------------------+

```


## 7. OPA run（服务器）
要与OPA集成，您可以将其作为服务器运行并通过HTTP执行查询。您可以使用-s或将OPA作为服务器启动`--server`：

```bash
./opa run --server ./example.rego
```
默认情况下，OPA在上侦听HTTP连接`0.0.0.0:8181`。请参阅参考资料`opa run --help`，以获取用于更改侦听地址，启用TLS等的选项的列表。

在另一个终端内部使用curl（或类似的工具）来访问OPA的`HTTP API`。查询/v1/dataHTTP API时，必须将输入数据包装在JSON对象内：

```bash
{
    "input": <value>
}
```
创建输入文件的副本，以通过发送curl：

```bash
cat <<EOF > v1-data-input.json
{
    "input": $(cat input.json)
}
EOF
```
执行一些curl请求并检查输出：

```bash
curl localhost:8181/v1/data/example/violation -d @v1-data-input.json -H 'Content-Type: application/json'
curl localhost:8181/v1/data/example/allow -d @v1-data-input.json -H 'Content-Type: application/json'
```
默认情况下`data.system.main`，用于不带路径的策略查询。当您在不提供路径的情况下执行查询时，不必包装输入。如果该`data.system.main`决定未定义，则将其视为错误：

```bash
curl localhost:8181 -i -d @input.json -H 'Content-Type: application/json'
```

您可以重新启动OPA并将其配置为使用任何决策作为默认决策：

```bash
./opa run --server --set=default_decision=example/allow ./example.rego
```

curl从上面重新运行最后一个命令：

```bash
curl localhost:8181 -i -d @input.json -H 'Content-Type: application/json'
```


## 8. Rego 语法
OPA策略以称为Rego的高级声明性语言表示。Rego（发音为“ ray-go”）是专门为在复杂的分层数据结构上表达策略而构建的。有关Rego的详细信息，请参阅[策略语言文档](https://www.openpolicyagent.org/docs/latest/policy-language/)。

> below以下示例是交互式的！如果在包含服务器，网络和端口的上方编辑输入数据，则输出将在下面更改。同样，如果您在下面的示例中编辑查询或规则，则输出将更改。在通读本节时，请尝试更改输入，查询和规则，并观察输出的差异。

������也可以使用以下命令在您的计算机上本地运行它们opa eval，这是设置说明。


### 8.1 参考
当OPA评估策略时，它将查询中提供的数据绑定到名为的全局变量input。您可以使用.（点）运算符在输入中引用数据。

```bash
input.servers
```

```c
[
  {
    "id": "app",
    "ports": [
      "p1",
      "p2",
      "p3"
    ],
    "protocols": [
      "https",
      "ssh"
    ]
  },
  {
    "id": "db",
    "ports": [
      "p3"
    ],
    "protocols": [
      "mysql"
    ]
  },
  {
    "id": "cache",
    "ports": [
      "p3"
    ],
    "protocols": [
      "memcache"
    ]
  },
  {
    "id": "ci",
    "ports": [
      "p1",
      "p2"
    ],
    "protocols": [
      "http"
    ]
  },
  {
    "id": "busybox",
    "ports": [
      "p1"
    ],
    "protocols": [
      "telnet"
    ]
  }
]
```
要引用数组元素，可以使用熟悉的方括号语法：

```bash
input.servers[0].protocols[0]
```

```bash
"https"
```

> keys如果键包含以外的其他字符，则可以使用相同的方括号语法 [a-zA-Z0-9_]。例如input["foo~bar"]。

如果引用的值不存在，则OPA返回undefined。未定义表示OPA无法找到任何结果。

```bash
input.deadbeef
```

```bash
undefined decision
```
### 8.2 表达式（逻辑与）
要在Rego中制定政策决策，您要针对输入和其他数据编写表达式。

```bash
input.servers[0].id == "app"
```

```c
true
```
OPA包括一组内置函数，可用于执行常见操作，例如字符串操作，正则表达式匹配，算术，聚合等。

```bash
count(input.servers[0].protocols) >= 1
```

```c
true
```
有关现成的OPA支持的内置功能的完整列表，请参阅“[策略参考](https://www.openpolicyagent.org/docs/latest/policy-reference/)”页面。

多个表达式通过;（AND）运算符连接在一起。为了使查询产生结果，查询中的所有表达式必须为真或已定义。表达式的顺序无关紧要。

```bash
input.servers[0].id == "app"; input.servers[0].protocols[0] == "https"
```

```c
true
```
您可以;通过将表达式分成多行来省略（AND）运算符。以下查询与上一个查询具有相同的含义：

```c
input.servers[0].id == "app"
input.servers[0].protocols[0] == "https"
```

```bash
true
```
如果查询中的任何表达式都不为真（或未定义），则结果为未定义。在下面的示例中，第二个表达式为false：

```bash
input.servers[0].id == "app"
input.servers[0].protocols[0] == "telnet"
```

```bash
undefined decision
```


### 8.3 逻辑或
在查询中将多个表达式连接在一起时，您表示的是逻辑与。要在Rego中表达逻辑OR，您可以定义多个具有相同名称的规则。让我们来看一个例子。

想象一下，您想知道是否有任何服务器公开允许客户端外壳访问的协议。为了确定这一点，你可以定义声明了一个完整的规则 `shell_accessible`是`true`，如果任何服务器暴露"`telnet`"或"`ssh`" 协议：

```bash
package example.logical_or

default shell_accessible = false

shell_accessible = true {
    input.servers[_].protocols[_] == "telnet"
}

shell_accessible = true {
    input.servers[_].protocols[_] == "ssh"
}
```

```bash
{
    "servers": [
        {
            "id": "busybox",
            "protocols": ["http", "telnet"]
        },
        {
            "id": "web",
            "protocols": ["https"]
        }
    ]
}
```

```bash
shell_accessible

true
```

> defaultkeyword如果未定义具有相同名称的所有其他规则，则该关键字告诉OPA为该变量分配一个值。


当您将逻辑或与部分规则一起使用时，每个规则定义都会影响分配给变量的一组值。例如，可以将上面的示例修改为生成一组公开"telnet"或 的服务器"ssh"。


```bash
package example.logical_or

shell_accessible[server.id] {
    server := input.servers[_]
    server.protocols[_] == "telnet"
}

shell_accessible[server.id] {
    server := input.servers[_]
    server.protocols[_] == "ssh"
}
```

```bash
{
    "servers": [
        {
            "id": "busybox",
            "protocols": ["http", "telnet"]
        },
        {
            "id": "db",
            "protocols": ["mysql", "ssh"]
        },
        {
            "id": "web",
            "protocols": ["https"]
        }
    ]
}
```

```bash
shell_accessible

[
  "busybox",
  "db"
]
```

### 8.4 Variables变量

您可以使用`:=`（赋值）运算符将值存储在中间变量中。可以像一样引用变量`input`

```bash
s := input.servers[0]
s.id == "app"
p := s.protocols[0]
p == "https"
```

```c
+---------+-------------------------------------------------------------------+
|    p    |                                 s                                 |
+---------+-------------------------------------------------------------------+
| "https" | {"id":"app","ports":["p1","p2","p3"],"protocols":["https","ssh"]} |
+---------+-------------------------------------------------------------------+
```
当OPA评估表达式时，它将查找使所有表达式都为真的变量的值。如果没有使所有表达式都为真的变量赋值，则结果是不确定的。

```bash
s := input.servers[0]
s.id == "app"
s.protocols[1] == "telnet"
```

```bash
undefined decision
```
变量是不可变的。如果您尝试两次分配相同的变量，OPA将报告错误。

```bash
s := input.servers[0]
s := input.servers[1]
```

```bash
1 error occurred: 2:1: rego_compile_error: var s assigned above
```
OPA必须能够枚举所有表达式中所有变量的值。如果OPA无法枚举任何表达式中变量的值，则OPA将报告错误。

```bash
x := 1
x != y  # y has not been assigned a value
```

```c
2 errors occurred:
2:1: rego_unsafe_var_error: var _ is unsafe
2:1: rego_unsafe_var_error: var y is unsafe
```
### 8.5 迭代
像其他声明性语言（例如SQL）一样，Rego没有显式的循环或迭代构造。而是将变量注入表达式中时，隐式发生迭代。

要了解Rego中迭代的工作原理，请想象您需要检查是否有任何公共网络。回想一下，网络是在数组中提供的：

```bash
input.networks
```

```bash
[
  {
    "id": "net1",
    "public": false
  },
  {
    "id": "net2",
    "public": false
  },
  {
    "id": "net3",
    "public": true
  },
  {
    "id": "net4",
    "public": true
  }
]
```
一种选择是测试输入中的每个网络：

```bash
input.networks[0].public == true
false

input.networks[1].public == true
false

input.networks[2].public == true
true
```
这种方法是有问题的，因为可能有太多网络无法静态列出，或更重要的是，可能无法事先知道网络的数量。

在Rego中，解决方案是将数组索引替换为变量。

```bash
some i; input.networks[i].public == true
```

```bash
+---+
| i |
+---+
| 2 |
| 3 |
+---+
```
现在，查询将要求该值i使整个表达式为真。当您在引用中替换变量时，OPA会自动查找满足查询中所有表达式的变量分配。就像中间变量一样，OPA返回变量的值。

您可以根据需要替换任意多个变量。例如，要确定是否有服务器公开了不安全的"http"协议，您可以编写：

```bash
some i, j; input.servers[i].protocols[j] == "http"

+---+---+
| i | j |
+---+---+
| 3 | 0 |
+---+---+
```
如果变量出现多次，则分配满足所有表达式。例如，要查找连接到公共网络的端口的ID，可以编写：

```bash
some i, j
id := input.ports[i].id
input.ports[i].network == input.networks[j].id
input.networks[j].public
```

```bash
+---+------+---+
| i |  id  | j |
+---+------+---+
| 1 | "p2" | 2 |
+---+------+---+
```
为变量提供好名字可能很难。如果仅引用一次变量，则可以将其替换为特殊的_（通配符变量）运算符。从概念上讲，的每个实例_都是一个唯一变量。

```bash
input.servers[_].protocols[_] == "http"
true
```
就像引用不存在的字段或无法匹配的表达式的引用一样，如果OPA无法找到满足所有表达式的任何变量赋值，则结果是不确定的。

```bash
some i; input.servers[i].protocols[i] == "ssh"  # there is no assignment of i that satisfies the expression

undefined decision
```
### 8.6 规则
Rego使您可以使用规则封装和重用逻辑。规则只是if-then逻辑语句。规则可以是“完整”或“部分”。

#### 8.6.1 完整规则
完整的规则是if-then语句，这些语句将单个值分配给变量。例如：

```bash
package example.rules

any_public_networks = true {  # is true if...
    net := input.networks[_]  # some network exists and..
    net.public                # it is public.
}
```
每条规则都由一个头和一个身体组成。在Rego中，如果规则主体对于某些变量分配集为true，则说规则标题为true 。在上面的示例`any_public_networks = true`中，头是`net := input.networks[_]`; `net.public`是身体。

您可以查询规则生成的值，就像其他任何值一样：

```bash
any_public_networks
true
```
规则生成的所有值都可以通过全局data变量查询。

```bash
data.example.rules.any_public_networks
true
```

> 您可以通过使用绝对路径引用OPA加载的任何规则来查询其值。规则的路径始终为：
> `data.<package-path>.<rule-name>`。

如果您省略`= <value>`规则标题的一部分，则该值默认为`true`。您可以按以下方式重写上面的示例，而无需更改其含义：

```bash
package example.rules

any_public_networks {
    net := input.networks[_]
    net.public
}
```
要定义常量，请省略规则主体。省略规则正文时，默认为`true`。由于规则主体为true，因此规则标头始终为`true / defined`。

```bash
package example.constants

pi = 3.14
```

可以像其他任何值一样查询这样定义的常量：


```bash
pi > 3

true
```
如果OPA无法找到满足规则主体的变量分配，则可以说该规则是未定义的。例如，如果input提供给OPA的不包括公共网络，`any_public_networks`则将是未定义的（与false相同）。下面，为OPA提供了一组不同的输入网络（都不是公共的）：

```bash
{
    "networks": [
        {"id": "n1", "public": false},
        {"id": "n2", "public": false}
    ]
}
```

```bash
any_public_networks


undefined decision
```
#### 8.6.2 部分规则
部分规则是`if-then`语句，它们生成一组值并将该组值分配给变量。例如： 

```bash
package example.rules

public_network[net.id] {      # net.id is in the public_network set if...
    net := input.networks[_]  # some network exists and...
    net.public                # it is public.
}
```
在上面的示例中`public_network[net.id]`是规则头，并且`net := input.networks[_]`; `net.public`是规则主体。您可以像查询其他任何值一样查询整个值集：

```bash
public_network

[
  "net3",
  "net4"
]
```
您可以通过使用变量引用`set`元素来遍历值集：

```bash
some n; public_network[n]

+--------+-------------------+
|   n    | public_network[n] |
+--------+-------------------+
| "net3" | "net3"            |
| "net4" | "net4"            |
+--------+-------------------+
```
最后，您可以使用相同的语法检查集合中是否存在值：

```bash
public_network["net3"]

"net3"
```
除了部分定义集合外，您还可以部分定义键/值对（也称为对象）。有关更多信息，请参见 语言指南中的[规则](https://www.openpolicyagent.org/docs/latest/policy-language/#rules)。




### 8.7 语法示例
以上各节介绍了Rego的核心概念。综上所述，让我们回顾一下所需的策略（英语）：

1. Servers reachable from the Internet must not expose the insecure 'http' protocol.
从Internet可访问的服务器不能暴露不安全的“http”协议。
2. Servers are not allowed to expose the 'telnet' protocol.
服务器不允许公开'telnet'协议。

在较高级别，该策略需要识别违反某些条件的服务器。为了实施此策略，我们可以定义称为的规则violation ，这些规则生成一组违反的服务器。

例如：

```bash
package example

allow = true {                                      # allow is true if...
    count(violation) == 0                           # there are zero violations.
}

violation[server.id] {                              # a server is in the violation set if...
    some server
    public_server[server]                           # it exists in the 'public_server' set and...
    server.protocols[_] == "http"                   # it contains the insecure "http" protocol.
}

violation[server.id] {                              # a server is in the violation set if...
    server := input.servers[_]                      # it exists in the input.servers collection and...
    server.protocols[_] == "telnet"                 # it contains the "telnet" protocol.
}

public_server[server] {                             # a server exists in the public_server set if...
    some i, j
    server := input.servers[_]                      # it exists in the input.servers collection and...
    server.ports[_] == input.ports[i].id            # it references a port in the input.ports collection and...
    input.ports[i].network == input.networks[j].id  # the port references a network in the input.networks collection and...
    input.networks[j].public                        # the network is public.
}
```

```bash
some x; violation[x]

+-----------+--------------+
|     x     | violation[x] |
+-----------+--------------+
| "ci"      | "ci"         |
| "busybox" | "busybox"    |
+-----------+--------------+
```



## 9. 将 OPA 用作Go库
OPA可以作为库嵌入到Go程序中。将OPA嵌入为库的最简单方法是导入`github.com/open-policy-agent/opa/rego` 软件包。

```bash
import "github.com/open-policy-agent/opa/rego"
```
调用该`rego.New`函数以创建可以准备或评估的对象：

```bash
r := rego.New(
    rego.Query("x = data.example.allow"),
    rego.Load([]string{"./example.rego"}, nil))
```
支持多种选项自定义的评价。有关详细信息，请参见[GoDoc](https://pkg.go.dev/github.com/open-policy-agent/opa/rego)页面。构造新`rego.Rego`对象后，您可以调用 `PrepareForEval()`以获得可执行查询。如果`PrepareForEval()`失败，则表明传递给`rego.New()`调用的选项之一无效（例如，解析错误，编译错误等）

```bash
ctx := context.Background()
query, err := r.PrepareForEval(ctx)
if err != nil {
    // handle error
}
```
可以将准备好的查询对象缓存在内存中，在多个`goroutine`中共享，并使用不同的输入重复调用。调用`Eval()`以执行准备好的查询。

```bash
bs, err := ioutil.ReadFile("./input.json")
if err != nil {
    // handle error
}

var input interface{}

if err := json.Unmarshal(bs, &input); err != nil {
    // handle error
}

rs, err := query.Eval(ctx, rego.EvalInput(input))
if err != nil {
    // handle error
}
```
该策略决策包含在Eval()调用返回的结果中。您可以检查该决定并进行相应处理：

```bash
// In this example we expect a single result (stored in the variable 'x').
fmt.Println("Result:", rs[0].Bindings["x"])
```
您可以将上述步骤组合到一个简单的命令行程序中，该程序可以评估策略并输出结果：

main.go：

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"

	"github.com/open-policy-agent/opa/rego"
)

func main() {

	ctx := context.Background()

	// Construct a Rego object that can be prepared or evaluated.
	r := rego.New(
		rego.Query(os.Args[2]),
		rego.Load([]string{os.Args[1]}, nil))

	// Create a prepared query that can be evaluated.
	query, err := r.PrepareForEval(ctx)
	if err != nil {
		log.Fatal(err)
	}

	// Load the input document from stdin.
	var input interface{}
	dec := json.NewDecoder(os.Stdin)
	dec.UseNumber()
	if err := dec.Decode(&input); err != nil {
		log.Fatal(err)
	}

	// Execute the prepared query.
	rs, err := query.Eval(ctx, rego.EvalInput(input))
	if err != nil {
		log.Fatal(err)
	}

    // Do something with the result.
	fmt.Println(rs)
}
```
运行以下代码，如下所示：

```bash
go run main.go example.rego 'data.example.violation' < input.json
[{[[ci busybox]] map[]}]
```
参考:

 - [openpolicyagent官网](https://www.openpolicyagent.org/docs/latest/)
 - [Open Policy Agent: What Is OPA and How It Works (Examples)](https://spacelift.io/blog/what-is-open-policy-agent-and-how-it-works)
 - [Open Policy Agent: Authorization in a Cloud Native World](https://www.aquasec.com/cloud-native-academy/cloud-native-applications/open-policy-agent-authorization-in-a-cloud-native-world/)

