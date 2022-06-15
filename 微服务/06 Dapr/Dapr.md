# Dapr-分布式应用运行时

## 一、Dapr起源

### 1. 什么是Service Mesh

从 2010 年的时候，SOA 架构就已经在中大型互联网公司中开始流行，阿里也在2012 年开源了 Dubbo 。而之后微服务架构开始流行，大量互联网和传统企业都投身到微服务的建设中。在国内逐渐形成了Dubbo 和 Spring Cloud 两大微服务阵营。在2016 年的时候，微服务领域一个更具有前沿性，更加符合容器和 Kubernetes 的微服务方案正在孕育，这个技术被称为 Service Mesh。时至今日，Service Mesh 理念已经得到了大范围普及，很多公司都在 Service Mesh 领域有了落地。

#### 1.1  Service Mesh 定义

Service Mesh 是一个基础设施层，主要围绕服务间通信来进行。现在的云原生应用的服务拓扑结构非常复杂，Service Mesh 可以在这种复杂拓扑结构中实现可靠的请求传送。Service Mesh 是以 `Sidecar`的方式运行，应用旁边会运行一个独立的 Service Mesh 进程，Service Mesh 负责远程服务的通信。军用三轮摩托车和 Service Mesh 非常相像，军用三轮摩托车上一个士兵负责开车，一个士兵负责对人发起射击。

远程服务的通信：

- 服务注册与发现
- 服务之间调用
- 服务之间降级，熔断
- 服务之间链接追踪
- 服务监控

#### 1.2  Service Mes解决的问题

![image-20220415093040095](images/image-20220415093040095.png)

**传统的微服务架构**大都以 RPC 通信框架为基础，在 RPC SDK 中提供了服务注册/发现，服务路由，负载均衡，全链路跟踪等能力。应用业务逻辑和 RPC SDK 在同一个进程中，这种方式给传统微服务架构带了很多挑战：中间件能力相关代码侵入到了业务代码中，耦合性很高；推动 RPC SDK 的升级成本非常高，进而也导致了 SDK 版本分化非常严重。同时这种方式对应用开发者的要求比较高，需要有丰富的服务治理的运维能力，有中间件的背景知识，使用中间件的门槛偏高。

通过 **Service Mesh 方式**将一些 RPC 的能力进行下沉，这样可以很好的实现关注点分离、职责边界的明确。随着容器和 Kubernetes 技术的发展，Service Mesh 已经成为云原生的基础设施。

#### 1.3 Service Mesh小结

- Service Mesh 定位就是提供服务间通信的基础设施，社区里主要支持 RPC 和http 。
- 采用 Sidecar 方式部署，支持部署在 Kubernetes 和虚拟机之上。
- Service Mesh 采用原协议转发，所以 Service Mesh 也被称为网络代理。正是由于这种方式方式，所以可以做到对应用的零侵入。

### 2 Dapr的诞生

#### 2.1 Service Mesh 遇到的挑战

![image-20210815103433402](images/image-20210815103433402.png)

用户在云上部署业务的形态主要有普通应用类型和FaaS类型。Faas （功能服务化）场景下，比较吸引用户的是成本和研发效率，成本主要通过按需分配和极致的弹性效率来达成。而应用开发者期望通过 FaaS 提供多语言的编程环境，提升研发效率，包括启动时间、发布时间、开发的效率。
​

Service Mesh 的实现，本质是原协议转发，原协议转发可以给应用带来零侵入的优势。但是原协议转发也带来了一些问题，应用侧中间件SDK还需要去实现序列化和编解码工作，所以在多语言实现方面还有一定成本；随着开源技术的不断发展，使用的技术也在不断迭代，如果想从 Spring Cloud 迁移到 Dubbo ，要么应用开发者需要切换依赖的 SDK，如果想借助Service Mesh来达到这个效果，Service Mesh 需要进行协议转换，成本较高。
​

Service Mesh 更加聚焦于服务间的通讯，而对其他形态的 Mesh 的支持上非常少。比如 Envoy， 除了在 RPC 领域比较成功外，在 Redis、消息等领域的尝试都未见成效。蚂蚁的 Mosn 中支持了 RPC 和消息的集成。整体多 Mesh 形态的需求是存在的，但是各个 Mesh 产品各自发展，缺少抽象和标准。如此多形态的 Mesh ，是共用一个进程吗？如果是共用一个进程，那么是共用一个端口吗？许多问题都没有答案。而控制面方面，从功能角度来看的话，大都围绕流量来展开。看过 xDS 协议里的内容，核心是围绕发现服务和路由来展开。其他类型的分布式能力，在 Service Mesh的控制面中基本没有涉及，更谈不上抽象各种类似 xDS 的协议去支持这些分布式能力。
​

因为成本和研发效率等原因，FaaS 受到了越来越多客户的选择，FaaS 对多语言和编程 API 的友好性上有了更多诉求，那么 Service Mesh 在这两块还是不能给客户带来额外的的价值。

#### 2.2 Multiple Runtime 

![image-20220415105554741](images/image-20220415105554741.png)

Bilgin Ibryam 是 Kubernetes Patterns 的作者，是 RedHat 的首席中间件架构师，在 Apache 社区里非常活跃。他发表了一篇文章对当前分布式的一些困难和问题进行了抽象，将分布式应用需求分成了 4 个大种类：生命周期、网络、状态、绑定。每种类型下面还有一些子能力，如 Point-to-Point， pub/sub， Caching 等比较经典的中间件能力。应用对分布式能力有如此多的需求，而 Service Mesh 显然不能满足应用的当前的需求。Biligin Ibryam 还在文章中提出了 Multiple Runtime 的理念来解决Service Mesh 的困境

分布式应用的需求:

- 生命周期：包括部署，健康检查，水平扩展，配置管理等，目前这些需求的最佳实践，都陆续在 kubernetes 上有了落地。
- 网络：网络方面的需求 是 service Mesh 的主战场，比如 istio 可以满足这里绝大部分需求，除了 pub/sub。
- 状态：包括数据的读写，状态其实是非常难以管理的，涉及幂等，缓存，数据流等等。
- 绑定：主要是指和系统外部资源的交互。

![image-20220415105908644](images/image-20220415105908644.png)

在传统的中间件模式下，应用和分布式能力是在一个进程中，以 SDK 方式进行集成。随着各种基础设施下沉，各种分布式能力从应用中移到了应用外。如 K8s 负责了生命周期相关的需求，Istio、Knative 等都负责一些分布式能力。如果将这些能力都移动到独立的 Runtime 中，那么这种情况无论从运维层面还是资源层面来看，都是没办法接受的。所以这时候肯定需要将部分 Runtime 进行整合，最理想的方式肯定是整合成一个。这种方式被定义成 Mecha ，中文意思是机甲的意思，就像日本动漫里主人公变身穿上机甲，机甲的每个部件就像一个分布式能力，机甲里的人对应的是主应用，也叫 Micrologic Runtime 。 这两个 Runtime 可以是一对一的 Sidecar 方式，这种非常适合传统的应用；也可以是多对一的 Node 模式，适合边缘场景或者网管模式下。

那么对于将各种分布式能力进行整合的 Mecha Runtime 这一目标本身问题不大，那么怎么整合呢？对 Mecha 有什么要求呢？

- Mecha 的组件能力是抽象的，任何一个开源产品可以快速进行扩展和集成。
- Mecha 需要有一定的可配置能力，可以通过 yaml/json 进行配置和激活。这些文件格式最好能和主流的云原生方式对
- Mecha 提供标准的 API ，和主应用之间的交互的网络通信基于此 API 来完成，不再是原协议转发，这样对于组件扩展和 SDK 的维护都能带来极大的便利性。
- 分布式能力中的生命周期，可以将部分能力交接过底层的基础设施，比如 K8s。当然有些复杂的场景，可能需要 K8s、APP、Mecha Runtime 一起来完成。

既然最理想只剩下一个 Runtime , 那么为什么还叫 Multiple Runtime 呢？因为应用本身其实也是一个 Runtime ，再加上 Mecha Runtime ，所以至少是两个 Runtime 。

#### 2.3 分布式运行时

提供分布式应用运行所需要的环境

![image-20220419183112784](images/image-20220419183112784.png)

## 二、Dapr简介

### 1. Dapr介绍

前面的 Multiple Runtime 介绍地比较抽象，可以来从 Dapr 来重新理解下 Multiple Runtime 。Dapr 是 Multiple Runtime 的一个很好的践行者，所以 Dapr 肯定和应用共存的，要么是 Sidecar 模式，要么是 Node 模式。Dapr 这个词其实是不是造出来的，而是 Distributed Application Runtime 的首字母拼接而成，Dapr 这个图标可以看出来是一个帽子，这个帽子其实是一个服务生的帽子，表示的含义是要为应用做好服务。

![image-20220415110333299](images/image-20220415110333299.png)

从上图我们可以看到，使用Dapr开发的微服务应用理论上可以部署到任意的云提供商、边缘设备、虚拟机、Service Fabric集群、Kubernetes集群里面。

并且在如下方面（称之为Building Block）进行外部依赖的抽象：

- 服务之间的调用：不用操心服务注册和发现的问题，也不用操心重试等问题
- 状态的存储：通过键值对来存储应用逻辑的状态，让无状态服务变为有状态服务
- 事件的发布订阅：微服务之间除了直接的远程调用外，更多是依赖事件驱动的模式去产生关系，这里对事件发布订阅进行了统一
- 外部资源的绑定：当需要把事件发布到微服务应用外部，或者从外部接收事件的时候，就可以使用资源绑定
- Actors模式（单线程串行处理模式）：通过Actors的模式，让微服务可以以单线程的代码实现大规模并行处理。实际上，Actors这部分功能的开发人员就是来自于Service Fabric团队，两者的API也基本保持一致。通过这样的模式，也把Actors这种模式引入到了其他运行平台
- 可观测性：Dapr会帮助微服务应用发出基于W3C Trace Context标准和Open Telemetry规范的度量、日志、跟踪信息，方便对应用进行调试、诊断和监控
- 密钥管理：为微服务提供密钥管理的能力，从而从依赖的外部密钥管理服务中独立出来。

### 2. 特性

![image-20220415110728078](images/image-20220415110728078.png)

应用开发者使用各种多语言的 Dapr SDK 就可以直接拥有各种分布式能力。当然开发者也可以自己基于 HTTP 和 gRPC 来完成调用。Dapr 可以运行在大部分环境里，包括你自己电脑的环境，或者任何 Kubernetes 环境下，或者边缘计算场景，或者阿里云、AWS、GCP 等云厂商。

Dapr 社区里已经集成了 70+ 的 components 实现，应用开发者可以快速进行选择和使用。相似能力的组件的替换，可以通过 Dapr 里完成，应用侧可以做到无感知。

### 3. 核心模块

![image-20220415110758061](images/image-20220415110758061.png)

我们从 Dapr 产品模块纬度来解析下，看为什么 Dapr 是 Mulitiple Runtime 的一个很好实践。

Component 机制确保了可以快速扩展能力的实现，现在社区已经有的 Components实现已经有 70 个以上，不只包含开源产品，还包含云上的商业化产品。
​

Building Block 表示的的分布式能力，现在只支持 7 个，后续需要更多的分布式能力能够进来。BuildingBlock 现在支持了 HTTP 和 gRPC 这两种开放，而且普及度已经非常高的协议。而 Dapr 中 Building Block 下具体那些 Components 会被激活，需要依赖 YAML 文件来进行。正因为 Dapr 中采用了 HTTP、gRPC 的方式暴露能力，所以在应用侧想要支持多语言的标准的API编程界面就变得更为容易了。

### 4. 架构

![image-20220419184522261](images/image-20220419184522261.png)

## 三、Dapr环境部署（单机）

### 1. 部署Dapr单机环境

（1） 配置DNS

> 进入网站https://ipaddress.com/website/github.com 查询域名对应IP进行本地DNS解析配置
>
> 140.82.113.3 github.com
> 185.199.111.153 assets-cdn.github.com
> 199.232.69.194 github.global.ssl.fastly.net

修改vim /etc/hosts

![image-20220415154718116](images/image-20220415154718116.png)



（2） 安装Dapr环境（本次使用版本：V1.7.0）

- 官方安装方法如下

```shell
wget -q https://raw.githubusercontent.com/dapr/cli/master/install/install.sh -O - | /bin/bash
```

说明：同样因为墙的问题，我们无法在Linux上直接在线安装CLI，可以通过官方文档途径下载dapr_linux_amd64.tar.gz包安装。

下载后解压到/usr/local/bin目录即可

输入dapr命令验证CLI是否安装成功

- 国内安装方法如下：

https://github.com/dapr/cli/releases

说明：需要使用VPN，否则下载时会提示“无法下载-需要授权”

![image-20220415155413186](images/image-20220415155413186.png)

```shell
# 解压文件到/usr/loacl/bin(必须解压到该文件下)
tar -zxvf dapr_linux_amd64.tar.gz -C /usr/local/bin

# 测试是否安装成功
dapr -v
```



（3） 初始化Dapr

```
dapr init
```

![image-20220418111734983](images/image-20220418111734983.png)

说明：

Dapr初始化成功后，会在用户文件夹下会生成.dapr/bin文件夹，如/root/.dapr/bin：

Web：Dapr Dashbor的，UI显示

Daprd：Dapr运行环境

![image-20220418112003347](images/image-20220418112003347.png)

（4） 验证是否成功

默认初始化了三个docker容器

![image-20220418112159520](images/image-20220418112159520.png)

验证版本是否匹配

![image-20220418135358206](images/image-20220418135358206.png)

### 2. 核心组件

#### 2.1 服务调用

准备工作：创建backend、frontend WebAPI

![image-20210814162525996](images/image-20210814162525996.png)

##### 2.1.1 Dapr的RPC调用

**Frontend项目**

（1）新增Nuget引用

​		Dapr.AspNetCore

（2）注册Dapr

​		services.AddControllers().AddDapr();

（3）创建控制器DaprController

```csharp
 [ApiController]
    [Route("[controller]")]
    public class DaprController : ControllerBase
    {
        private readonly ILogger<DaprController> _logger;
        private readonly DaprClient _daprClient;

        public DaprController(ILogger<DaprController> logger, DaprClient daprClient)
        {
            _logger = logger;
            _daprClient = daprClient;
        }
        /// <summary>
        /// 新增API，通过HttpClient调用Backend
        /// </summary>

        [HttpGet]
        public async Task<ActionResult> HttpClientAsync()
        {
            using var httpClient = DaprClient.CreateInvokeHttpClient();
            var result = await httpClient.GetAsync("http://backend/WeatherForecast");
            var resultContent = string.Format("result is {0} {1}", result.StatusCode, await result.Content.ReadAsStringAsync());
            return Ok(resultContent);
        }
        /// <summary>
        /// 新增接口，通过DaprClient调用BackEnd
        /// </summary>
        /// <returns></returns>
        [HttpGet("dapr_client")]
        public async Task<ActionResult> DaprClientAsync()
        {
            using var daprClient = new DaprClientBuilder().Build();
            var result = await daprClient.InvokeMethodAsync<IEnumerable<WeatherForecast>>(HttpMethod.Get, "backend", "WeatherForecast");
            return Ok(result);
        }
        /// <summary>
        /// 新增接口，通过注入的daprclient，调用BackEnd
        /// </summary>
        /// <returns></returns>
        [HttpGet("inject_dapr_client")]
        public async Task<ActionResult> InjectDaprAsync()
        {
            var result = await _daprClient.InvokeMethodAsync<IEnumerable<WeatherForecast>>(HttpMethod.Get, "backend", "WeatherForecast");
            return Ok(result);
        }
    }
```

（4）上传本地发布项目文件到Linux服务器

```shell
#运行frontend服务
dapr run --dapr-http-port 3512 --app-port 5020 --app-id frontend -- dotnet run --urls="http://*:5020"
#运行backend服务
dapr run --dapr-http-port 3513 --app-port 5030 --app-id backend -- dotnet run --urls="http://*:5030"

```

（5）验证测试

访问：http://192.168.200.104:5020/swagger/index.html

![image-20220418174630768](images/image-20220418174630768.png)

以上接口返回OK，则成功

##### 2.1.2 Dapr的GRPC调用

（1）创建daprgrpc项目

（2）添加Nuget包

> Dapr.AspNetCore
>
> <PackageReference Include="Dapr.AspNetCore" Version="1.6.0" />
>
> Grpc.AspNetCore
>
>  <PackageReference Include="Grpc.AspNetCore" Version="2.32.0" />

（3）修改daprgrpc项目

① greet.proto

```csharp
syntax = "proto3";

option csharp_namespace = "daprgrpc";

package greet;


// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings.
message HelloReply {
  string message = 1;
}

```

② 追加Service/HelloService.cs

```c#
    public class HelloService : AppCallback.AppCallbackBase
    {
        public override async Task<InvokeResponse> OnInvoke(InvokeRequest request, ServerCallContext context)
        {
            var response = new InvokeResponse();
            switch (request.Method)
            {
                case "grpc":
                    var input = request.Data.Unpack<HelloRequest>();
                    response.Data = Any.Pack(new HelloReply { Message = "ok" });
                    break;
            }
            return response;
        }
    }
```

③ Starrup.cs新增终结点映射

```c#
endpoints.MapGrpcService<HelloService>();
```

（4）修改frontend项目

① 追加依赖

><PackageReference Include="Google.Protobuf" Version="3.20.0" />
><PackageReference Include="Grpc.Net.Client" Version="2.44.0" />
><PackageReference Include="Grpc.Tools" Version="2.45.0">

② 追加Protos文件

直接将daprgrpc中的protos文件拷贝过来即可，然后修改GrpcServices类型为Client

> <Protobuf Include="Protos\greet.proto" GrpcServices="Client" />

③追加DaprGrpcController.cs文件

```C#
    [Route("api/[controller]")]
    [ApiController]
    public class DaprGrpcController : ControllerBase
    {
        private readonly DaprClient _daprClient;
        public DaprGrpcController(DaprClient _daprClient)
        {
            this._daprClient = _daprClient;
        }

        [HttpGet("grpc")]
        public async Task<ActionResult> GrpcAsync()
        {
            var result = await _daprClient.InvokeMethodGrpcAsync<HelloRequest, HelloReply>("grpcbackend", "grpc", new HelloRequest { Name = "Gerry Grpc" });
            return Ok(result);
        }
    }
```

（5）上传本地发布项目文件到Linux服务器

```shell
#运行frontend服务
dapr run --dapr-http-port 3512 --app-port 5020 --app-id frontend -- dotnet run --urls="http://*:5020"
#运行grpcbackend服务
dapr run --dapr-http-port 3514 --app-port 5040 --app-id grpcbackend --app-protocol  grpc -- dotnet run --urls="http://*:5040"
```

（5）验证测试

http://192.168.200.104:5020/swagger/index.html

![image-20220418190935702](images/image-20220418190935702.png)

/api/DaprGrpc/grpc 接口返回OK，则成功。

##### 2.1.3 服务调用原理

![image-20220418191119220](images/image-20220418191119220.png)

- 服务 A 对服务 B 发起HTTP/gRPC的调用。
- Dapr使用在给定主机平台上运行的名称解析组件发现服务B的位置。
- Dapr 将消息转发至服务 B的 Dapr 边车

**注**: Dapr 边车之间的所有调用考虑到性能都优先使用 gRPC。 仅服务与 Dapr 边车之间的调用可以是 HTTP 或 gRPC

- 服务 B的 Dapr 边车将请求转发至服务 B 上的特定端点 (或方法) 。 服务 B 随后运行其业务逻辑代码。
- 服务 B 发送响应给服务 A。 响应将转至服务 B 的边车。
- Dapr 将消息转发至服务 A 的 Dapr 边车。
- 服务 A 接收响应

#### 2.2 状态管理

##### 2.2.1 状态组件配置文件

默认如下（/root/.dapr/components/statestore.yaml）

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: localhost:6379
  - name: redisPassword
    value: ""
  - name: actorStateStore
    value: "true"
```

##### 2.2.2 项目实例

追加StateController.cs文件

```C#
    [Route("api/[controller]")]
    [ApiController]
    public class StateController : ControllerBase
    {
        private const string STATE_STORE = "statestore";//用于配置状态管理（需和statestore.yaml中的name保持一致）
        private const string KEY_NAME = "guid";
        private readonly ILogger<StateController> _logger;
        private readonly DaprClient _daprClient;
        public StateController(ILogger<StateController> logger, DaprClient daprClient)
        {
            _logger = logger;
            _daprClient = daprClient;
        }

        // 获取一个值
        [HttpGet]
        public async Task<ActionResult> GetAsync()
        {
            var result = await _daprClient.GetStateAsync<string>(STATE_STORE, KEY_NAME);
            return Ok(result);
        }

        //保存一个值
        [HttpPost]
        public async Task<ActionResult> PostAsync()
        {
            await _daprClient.SaveStateAsync<string>(STATE_STORE, KEY_NAME, Guid.NewGuid().ToString(), new StateOptions() { Consistency = ConsistencyMode.Strong });
            return Ok("done");
        }

        //删除一个值
        [HttpDelete]
        public async Task<ActionResult> DeleteAsync()
        {
            await _daprClient.DeleteStateAsync(STATE_STORE, KEY_NAME);
            return Ok("done");
        }

        //通过tag防止并发冲突，保存一个值
        [HttpPost("withtag")]
        public async Task<ActionResult> PostWithTagAsync()
        {
            var (_, etag) = await _daprClient.GetStateAndETagAsync<string>(STATE_STORE, KEY_NAME);
            await _daprClient.TrySaveStateAsync(STATE_STORE, KEY_NAME, Guid.NewGuid().ToString(), etag);
            return Ok("done");
        }

        //通过tag防止并发冲突，删除一个值
        [HttpDelete("withtag")]
        public async Task<ActionResult> DeleteWithTagAsync()
        {
            var (_, etag) = await _daprClient.GetStateAndETagAsync<string>(STATE_STORE, KEY_NAME);
            return Ok(await _daprClient.TryDeleteStateAsync(STATE_STORE, KEY_NAME, etag));
        }


        // 从绑定获取一个值，健值name从路由模板获取
        [HttpGet("frombinding/{name}")]
        public ActionResult GetFromBindingAsync([FromState(STATE_STORE, "name")] StateEntry<string> state)
        {
            return Ok(state.Value);
        }


        // 根据绑定获取并修改值，健值name从路由模板获取
        [HttpPost("withbinding/{name}")]
        public async Task<ActionResult> PostWithBindingAsync([FromState(STATE_STORE, "name")] StateEntry<string> state)
        {
            state.Value = Guid.NewGuid().ToString();
            return Ok(await state.TrySaveAsync());
        }


        // 获取多个个值
        [HttpGet("list")]
        public async Task<ActionResult> GetListAsync()
        {
            var result = await _daprClient.GetBulkStateAsync(STATE_STORE, new List<string> { KEY_NAME }, 10);
            return Ok(result);
        }

        // 删除多个个值
        [HttpDelete("list")]
        public async Task<ActionResult> DeleteListAsync()
        {
            var data = await _daprClient.GetBulkStateAsync(STATE_STORE, new List<string> { KEY_NAME }, 10);
            var removeList = new List<BulkDeleteStateItem>();
            foreach (var item in data)
            {
                removeList.Add(new BulkDeleteStateItem(item.Key, item.ETag));
            }
            await _daprClient.DeleteBulkStateAsync(STATE_STORE, removeList);
            return Ok("done");
        }
    }
```

上传本地发布项目文件到Linux服务器,并运行

```shell
dapr run --dapr-http-port 3512 --app-port 5020 --app-id frontend -- dotnet run --urls "http://*:5020"
```

访问：http://192.168.200.104:5020/swagger/index.html进行测试

##### 2.2.3 切换组件

①切换状态组件为MySQL，追加statestore-mysql.yaml

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore-mysql #状态管理名
spec:
  type: state.mysql
  version: v1
  metadata:
  - name: connectionString
    value: "root:123456@tcp(192.168.200.104:3306)/?allowNativePasswords=true"
```

② Docker部署MySQL

```shell
 docker run -itd --name mysql-test -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql
```



③ 修改StateController.cs文件

```C#
//private const string STATE_STORE = "statestore";//用于配置状态管理（需和statestore.yaml中的name保持一致）
private const string STATE_STORE = "statestore-mysql";//用于配置状态管理
```

④ 测试

访问：http://192.168.200.104:5020/api/State（追加key-value值）

MySQL已经插入了数据

![image-20220419104659597](images/image-20220419104659597.png)

##### 2.2.4 工作原理

应用程序可以使用Dapr的状态管理API，使用状态存储组件保存和读取键/值对。通过 HTTP 或 gRPC 调用 API。例如，通过使用HTTP POST可以保存键/值对，通过使用HTTP GET可以读取一个键并返回它的值。

![image-20220419105119296](images/image-20220419105119296.png)

#### 2.3 发布订阅

##### 2.3.1 发布订阅配置文件

默认如下（/root/.dapr/components/statestore.yaml）

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.redis
  version: v1
  metadata:
  - name: redisHost
    value: localhost:6379
  - name: redisPassword
    value: ""
```

##### 2.3.2 项目实例

①frontend项目追加PubController.cs

```C#
    [Route("api/[controller]")]
    [ApiController]
    public class PubsubController : ControllerBase
    {
        private DaprClient _daprClient;
        private ILogger<PubsubController> _logger;
        private string PUB_SUN = "pubsub";//配置发布订阅组件（需和pubsub.yaml中的name保持一致）
        private string TOPIC_NAME = "user_topic";

        public PubsubController(DaprClient daprClient, ILogger<PubsubController> logger)
        {
            _daprClient = daprClient;
            _logger = logger;
        }

        /// <summary>
        /// 发布消息的方法
        /// </summary>
        /// <returns></returns>
        [HttpPost]
        [Route("pub")]
        public async Task<IActionResult> PublishMessage()
        {
            _logger.LogInformation(">>>>>>>>>>>>>>>>>>>>>> 发布消息");
            var data = new UserInfo(40001, "张三", 43);
            await _daprClient.PublishEventAsync(PUB_SUN, TOPIC_NAME, data);

            return Ok("发布消息成功~");
        }
    }
    public record UserInfo(int UserId, string UserName, int Age);
```



②backend项目追加SubController.cs

```C#
    [ApiController]
    [Route("[controller]")]
    public class SubController : ControllerBase
    {
        private ILogger<SubController> _logger;

        public SubController(ILogger<SubController> logger)
        {
            _logger = logger;
        }


        [HttpPost("sub")]
        [Topic("pubsub", "user_topic")]
        public IActionResult ConsumerMessage(UserInfo user)
        {
            _logger.LogInformation(">>>>>>>>>>>>>>>>>>>>>>>消费消息");
            Console.WriteLine($"userId:{user.UserId} userName:{user.UserName}");
            return Ok();
        }
    }
    public record UserInfo(int UserId, string UserName, int Age);
```

**修改StartUp.cs文件（只需在订阅端添加如下代码）**

```C#
 app.UseCloudEvents();
 //添加订阅映射终端
 endpoints.MapControllers();
```



③测试

访问http://192.168.200.104:5020/api/Pubsub/pub（发布消息）

![image-20220419112316064](images/image-20220419112316064.png)

backend订阅侧，则会收到以下消息

![image-20220419112438971](images/image-20220419112438971.png)



##### 2.3.3 切换组件

①部署RabbitMQ组件

```shell
docker run -it -d --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.9-management
```

②新增pubsub-rabbitmq.yaml

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub-rab
  namespace: default
spec:
  type: pubsub.rabbitmq
  version: v1
  metadata:
  - name: host
    value: "amqp://guest:guest@192.168.3.254:5672"
  - name: durable
    value: "false"
  - name: deletedWhenUnused
    value: "false"
  - name: autoAck
    value: "false"
  - name: deliveryMode
    value: "0"
  - name: requeueInFailure
    value: "false"
  - name: prefetchCount
    value: "0"
  - name: reconnectWait
    value: "0"
  - name: concurrencyMode
    value: parallel
  - name: backOffPolicy
    value: "exponential"
  - name: backOffInitialInterval
    value: "100"
  - name: backOffMaxRetries
    value: "16"
```

③修改PubController.cs

```C#
  //private string PUB_SUN = "pubsub";//配置发布订阅组件（需和pubsub.yaml中的name保持一致）
  private string PUB_SUN = "pubsub-rab";
```

④修改SubController.cs

```C#
  //[Topic("pubsub", "user_topic")]
  [Topic("pubsub-rab", "user_topic")]
```

⑤ 测试

访问：http://192.168.200.104:5020/api/DaprGrpc/grpc（发布消息）

RabbitMQ已经生成该主题user-topic

![image-20220419135836453](images/image-20220419135836453.png)

##### 2.3.4 发布订阅原理

![image-20220419140407064](images/image-20220419140407064.png)

- 服务B的Dapr边车调用服务B（订阅者）的终结点`/dapr/subscribe` 。 服务会返回它要创建的订阅作为响应。
- 服务 B 的 Dapr 边车在消息代理上创建要求的订阅。
- 服务 A 在其Dapr边车的终结点`/v1.0/publish/<pub-sub-name>/<topic>` 上发布一条消息。
- 服务 A 的Dapr边车将消息发布到消息代理。
- 消息代理发送消息副本到服务 B 的Dapr边车。
- 服务 B 的Dapr边车调用服务 b 上与订阅对应的终结点（此处为 /orders）。服务以 HTTP 状态码 `200 OK` 进行响应，边车将认为消息已被成功处理。

#### 2.4 组件绑定

> Dapr 资源绑定需要通过**yaml文件**定义绑定组件。 此 YAML 文件描述要与其绑定的资源类型。 配置后，你的服务可以接收来自资源的事件或触发事件。

##### 2.4.1 输入绑定

输入绑定通过外部资源的传入事件触发代码。 下图的示例中需要在app中保留/tweet接口，以供sidecar调用.

![image-20220419192619735](images/image-20220419192619735.png)

- Dapr Sidecar读取绑定配置文件并订阅为外部资源。 
- 当外部资源触发时，在 Dapr sidecar中运行的绑定组件会选取它并触发一个事件。
- Dapr sidecar调用指定的接口。 在此示例中，服务在 `/tweet` 端口6000上侦听终结点上的 HTTP POST。 由于它是 HTTP POST 操作，因此在请求正文中传递事件的 JSON 有效负载。
- 处理事件后，服务将返回 HTTP 状态代码 `200 OK` 。

##### 2.4.2 输出绑定

输出绑定使服务能够触发调用外部资源。 跟输入绑定同样，需要配置描述输出绑定的绑定配置 YAML 文件。该事件在应用程序的 Dapr Sidecar上调用bingdings API。

![image-20220419192706025](images/image-20220419192706025.png)

- Dapr sidecar读取绑定配置文件，其中包含有关如何连接到外部资源的信息。
- 应用程序调用sidecar的 `/v1.0/bindings/sms` Dapr 终结点。 在这种情况下，它使用 HTTP POST 来调用 API。 还可以使用 gRPC。
- Dapr sidecar的绑定组件会调用外部消息系统来发送消息。 消息将包含 POST 请求中传递的负载。

##### 2.4.3 组件绑定配置文件

此处我们使用**rabbitmq**作为外部资源。因为rabbitmq*既支持输入绑定又支持输出绑定*。

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: RabbitBinding
spec:
  type: bindings.rabbitmq
  version: v1
  metadata:
  - name: queueName
    value: queue1
  - name: host
    value: amqp://admin:123456@192.168.200.104:5672
  - name: durable
    value: true
  - name: deleteWhenUnused
    value: false
  - name: ttlInSeconds
    value: 60
  - name: prefetchCount
    value: 0
  - name: exclusive
    value: false
  - name: maxPriority
    value: 5
```

##### 2.4.4 项目实例

###### 2.4.4.1 输入绑定

① Frontend追加RabbitBindingController.cs

```C#
    [Route("[controller]")]
    [ApiController]
    public class RabbitBindingController : ControllerBase
    {
        private readonly ILogger<RabbitBindingController> _logger;
        public RabbitBindingController(ILogger<RabbitBindingController> logger)
        {
            _logger = logger;
        }
        [HttpPost]
        public async Task<ActionResult> Post()
        {
            using (var reader = new StreamReader(Request.Body, System.Text.Encoding.UTF8))
            {
                var content = await reader.ReadToEndAsync();
                _logger.LogInformation(".............binding............." + content);
            }
            return Ok();
        }
    }
```

②启动RabbitMQ 

```shell
docker run -it -d --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.9-management
```



③启动Frontend程序

```shell
dapr run --dapr-http-port 3512 --app-port 5020 --app-id frontend -- dotnet run --urls="http://*:5020"
```



③ 然后打开RabbitMQ Management,查看Queue，发现queue1已成功创建

![image-20220420100733232](images/image-20220420100733232.png)



⑤测试

在RabbitMQ Management中直接发送消息

![image-20220420100846855](images/image-20220420100846855.png)

查看控制台日志，发现输入绑定成功触发

![image-20220420103249519](images/image-20220420103249519.png)

###### 2.4.4.2 输出绑定

① 修改Frontend/RabbitBindingController.cs类

```C#
    [Route("[controller]")]
    [ApiController]
    public class RabbitBindingController : ControllerBase
    {
        private readonly ILogger<RabbitBindingController> _logger;
        public RabbitBindingController(ILogger<RabbitBindingController> logger)
        {
            _logger = logger;
        }

        /// <summary>
        /// 输入绑定
        /// </summary>
        /// <returns></returns>
        //[HttpPost]
        //public async Task<ActionResult> Post()
        //{
        //    using (var reader = new StreamReader(Request.Body, System.Text.Encoding.UTF8))
        //    {
        //        var content = await reader.ReadToEndAsync();
        //        _logger.LogInformation(".............binding............." + content);
        //    }
        //    return Ok();
        //}

        /// <summary>
        /// 输出绑定
        /// </summary>
        /// <param name="daprClient"></param>
        /// <returns></returns>
        [HttpGet]
        public async Task<ActionResult> GetAsync([FromServices] DaprClient daprClient)
        {
            await daprClient.InvokeBindingAsync("RabbitBinding", "create", "测试，输入绑定");
            return Ok();
        }

    }
```

② 启动RabbitMQ 、启动Frontend程序

③ 测试

访问http://192.168.200.104:5020/RabbitBinding

![image-20220420103801278](images/image-20220420103801278.png)

查看RabbitMQ

![image-20220420112005045](images/image-20220420112005045.png)

**问题：**RabbiteMQ确实发布成功，但是被消费（Unacked表示消费后没有确认），原因不明。

![image-20220420112216475](images/image-20220420112216475.png)

#### 2.5 Actors

可用于解决秒杀问题

##### 2.5.1 Actor配置文件

statestore.yaml必须设置actorStateStore：true

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: localhost:6379
  - name: redisPassword
    value: ""
  - name: actorStateStore
    value: "true"
```



##### 2.5.2 工作原理

（1）Actor组件放置服务流程

![image-20220422154107466](images/image-20220422154107466.png)

- 启动时，Sidecar 调用执行组件服务以获取注册的执行组件类型和执行组件的配置设置。
- Sidecar 将注册的执行组件类型的列表发送到放置服务。
- 放置服务会将更新的分区信息广播到所有执行组件服务实例。 每个实例都将保留分区信息的缓存副本，并使用它来调用执行组件。

（2）调用Actor组件方法流程

![image-20220422154210474](images/image-20220422154210474.png)

- 服务在Sidecar 上调用执行组件 API。 请求正文中的 JSON 有效负载包含要发送到执行组件的数据。
- Sidecar 使用位置服务中的本地缓存的分区信息来确定哪个执行组件服务实例 (分区) 负责托管 ID 为的执行组件 `3` 。 在此示例中，它是 pod 2 中的服务实例。 调用将转发到相应的Sidecar 。
- Pod 2 中的Sidecar 实例调用服务实例以调用执行组件。 服务实例激活actor（如果它还没有激活）并执行actor 方法。

##### 2.5.3 项目实例

（1）引入依赖

```C#
<PackageReference Include="Dapr.Actors.AspNetCore" Version="1.6.0" />
```

（2）追加ISeckillActor

```C#
    public interface ISeckillActor : IActor
    {
        public Task Init();
        /// <summary>
        /// 获取库存
        /// </summary>
        /// <returns></returns>
        public Task<long> GetInventory();

        /// <summary>
        /// 下订单
        /// </summary>
        /// <returns></returns>
        public Task<long> CreateOrder();
    }
```

（3）追加SeckillActor

```C#
    public class SeckillActor : Actor, ISeckillActor
    {

        private readonly RedisHelper _redisHelper;

        public SeckillActor(ActorHost host, RedisHelper redisHelper) : base(host)
        {
            this._redisHelper = redisHelper;
        }

        /// <summary>
        /// 初始化库存
        /// </summary>
        /// <returns></returns>
        public async Task Init()
        {
            Console.WriteLine("1" );
            await StateManager.SetStateAsync("init", "init");
            Console.WriteLine("2");
            await _redisHelper.GetDatabase().StringSetAsync("InventoryNum", 10);
            Console.WriteLine("3");
            await _redisHelper.GetDatabase().StringSetAsync("OrderNum", 0);
        }
        /// <summary>
        /// 获取剩余库存
        /// </summary>
        /// <returns></returns>
        public Task<long> GetInventory()
        {
            var result = Convert.ToInt64(_redisHelper.GetDatabase().StringGet("InventoryNum").ToString());
            Console.WriteLine("库存为:" + result);
            return Task.FromResult(result);
        }
        /// <summary>
        /// 下订单
        /// </summary>
        /// <returns></returns>
        public Task<long> CreateOrder()
        {
            var result = GetInventory().Result;
            if (result > 0)
            {
                Console.WriteLine(">>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>");
                // 添加订单数
                _redisHelper.GetDatabase().StringIncrement("OrderNum");
                // 更新库存数
                return _redisHelper.GetDatabase().StringDecrementAsync("InventoryNum");
            }

            return Task.FromResult(long.Parse("-1"));
        }

    }
```

（4）追加ActorsController

```C#
    [Route("api/[controller]")]
    [ApiController]
    public class ActorsController : ControllerBase
    {
        [HttpPost("Init")]
        public void Init()
        {
            var actorinit = GetSeckill_Actor(Guid.NewGuid().ToString());
            actorinit.Init().Wait();
        }

        // Dotnet-Netty

        [HttpPost("Seckill")]
        public string Seckill()
        {
            var actor = GetSeckill_Actor(Guid.NewGuid().ToString());
            actor.CreateOrder();
            return "ok";
        }

        public static ISeckillActor actorid = GetSeckill_Actor("123");

        [HttpPost("SeckillActor")]
        public string SeckillActor1()
        {
            var actor = actorid;
            actor.CreateOrder();
            return "ok";
        }

        public static ISeckillActor GetSeckill_Actor(string actorid)
        {
            var actorId = new ActorId(actorid.ToString());
            return ActorProxy.Create<ISeckillActor>(actorId, nameof(SeckillActor));
        }
    }
```

（5）注册Actor，修改Satrtup.cs

```C#
   	//ConfigureServices
	services.AddActors(options => {
        // 注册Actor
        options.Actors.RegisterActor<TestActor>();
        options.Actors.RegisterActor<SeckillActor>();
     });
	//Configure
	endpoints.MapActorsHandlers();
```

（6）测试

**需要等frontend项目完全启动在执行以下接口，否则会如下报错**

![image-20220422153546098](images/image-20220422153546098.png)



![image-20220422152957994](images/image-20220422152957994.png)



①调用接口http://192.168.200.104:5020/api/Actors/Init进行初始化

②使用JMeter测试http://192.168.200.104:5020/api/Actors/Seckill

![image-20220422153047151](images/image-20220422153047151.png)

③使用JMeter测试②使用JMeter测试http://192.168.200.104:5020/api/Actors/SeckillActor

![image-20220422153140121](images/image-20220422153140121.png)

备注：JMeter配置如下

![image-20220422153238218](images/image-20220422153238218.png)

![image-20220422153305499](images/image-20220422153305499.png)

## 四、kubernetes部署

1、准备功能

（1）搭建K8s集群环境

参照《Kubernetes详细教程》文档安装部署K8s集群环境

![image-20220427182443457](images/image-20220427182443457.png)

（2）Habor安装

参照《Docker仓库》文档部署镜像私服仓库

```shell
#登录镜像仓库
docker login 192.168.3.251:9999
username: 输入账户
password: 输入密码
```

（3）打包镜像

```shell
cd /test/project/
docker build -t fronten:v11 -f frontend/Dockerfile .
```

（4）上传镜像

```shell
docker login 192.168.200.104
username: 输入账户
password: 输入密码
```

待完成

## 十、扩展

### 1.1 Sidecar

### 1.2 Istio

### 1.3 FaaS\Iaas\Paas\Daas
