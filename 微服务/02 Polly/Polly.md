## 微服务组件之Polly

### 1. 服务雪崩

​	在微服务架构中，当中间某一层的服务故障，则会导致上层服务的请求堆积，系统硬盘/CPU资源耗尽，从而会导致上游一连串的服务故障，而导致服务雪崩。即一个服务失败，导致整条链路的服务都失败。

解决方式：

#### （1）服务降级

- 当下游的服务因为某种原因**响应过慢**，下游服务主动停掉一些不太重要的业务，释放出服务器资源，增加响应速度！
- 当下游的服务因为某种原因**不可用**，上游主动调用本地的一些降级逻辑，避免卡顿，迅速返回给用户！

#### （2）服务熔断

	- 当下游的服务因为某种原因突然**变得不可用**或**响应过慢**，上游服务为了保证自己整体服务的可用性，不再继续调用目标服务，直接返回，快速释放资源（或者上游主动调用本地的一些降级逻辑，避免卡顿，迅速返回给用户）。如果目标服务情况好转则恢复调用。

### 2. Polly组件

​	Polly是一个开源的弹性瞬态故障处理库、它可以在程序出现故障、超时，或者返回值达到某种条件的时候进行多种策略处理。比如重试、超时、降级、熔断等等。

#### （1）超时策略

```c#

                var memberJson = await Policy.TimeoutAsync(5, TimeoutStrategy.Pessimistic, (t, s, y) =>
                {
                    Console.WriteLine("超时了~~~~");
                    return Task.CompletedTask;
                }).ExecuteAsync(async () =>
                {
                    // 业务逻辑
                    using var httpClient = new HttpClient();
                    httpClient.BaseAddress = new Uri($"http://localhost:5000");
                    var memberResult = await httpClient.GetAsync("/api/polly/timeout");
                    memberResult.EnsureSuccessStatusCode();
                    var json = await memberResult.Content.ReadAsStringAsync();
                    Console.WriteLine(json);

                    return json;
                });
```

#### （2）重试策略

```c#

                //当发生 HttpRequestException 的时候触发 RetryAsync 重试，并且最多重试3次。
                var memberJson1 = await Policy.Handle<HttpRequestException>().RetryAsync(3).ExecuteAsync(async () =>
                {
                    Console.WriteLine("重试中.....");
                    using var httpClient = new HttpClient();
                    httpClient.BaseAddress = new Uri($"http://localhost:8000");
                    var memberResult = await httpClient.GetAsync("/member/1001");
                    memberResult.EnsureSuccessStatusCode();
                    var json = await memberResult.Content.ReadAsStringAsync();

                    return json;
                });

                //使用 Polly 在出现当请求结果为 http status_code 500 的时候进行3次重试。
                var memberResult = await Policy.HandleResult<HttpResponseMessage>
                    (x => (int)x.StatusCode == 500).RetryAsync(3).ExecuteAsync(async () =>
                {
                    Thread.Sleep(1000);
                    Console.WriteLine("响应状态码重试中.....");
                    using var httpClient = new HttpClient();
                    httpClient.BaseAddress = new Uri($"http://localhost:5000");
                    var memberResult = await httpClient.GetAsync("/api/polly/error");

                    return memberResult;
                });
```

#### （3）降级策略

```c#

            //首先我们使用 Policy 的 FallbackAsync("FALLBACK") 方法设置降级的返回值。当我们服务需要降级的时候会返回 "FALLBACK" 的固定值。
            //同时使用 WrapAsync 方法把重试策略包裹起来。这样我们就可以达到当服务调用失败的时候重试3次，如果重试依然失败那么返回值降级为固定的 "FALLBACK" 值。
            var fallback = Policy<string>.Handle<HttpRequestException>().Or<Exception>().FallbackAsync("FALLBACK", (x) =>
            {
                Console.WriteLine($"进行了服务降级 -- {x.Exception.Message}");
                return Task.CompletedTask;
            }).WrapAsync(Policy.Handle<HttpRequestException>().RetryAsync(3));

            var memberJson = await fallback.ExecuteAsync(async () =>
            {
                using var httpClient = new HttpClient();
                httpClient.BaseAddress = new Uri($"http://localhost:5000");
                var result = await httpClient.GetAsync("/api/user/" + 1);
                result.EnsureSuccessStatusCode();
                var json = await result.Content.ReadAsStringAsync();
                return json;

            });
            Console.WriteLine(memberJson);
            if (memberJson != "FALLBACK")
            {
                var member = JsonConvert.DeserializeObject<User>(memberJson);
                Console.WriteLine($"{member!.Id}---{member.Name}");
            }
```

#### （4）熔断策略

#### （5）策略包裹

多种策略进行组装

```c#
            //定义熔断策略
            var circuitBreaker = Policy.Handle<Exception>().CircuitBreakerAsync(
               exceptionsAllowedBeforeBreaking: 2, // 出现几次异常就熔断
               durationOfBreak: TimeSpan.FromSeconds(10), // 熔断10秒
               onBreak: (ex, ts) =>
               {
                   Console.WriteLine("circuitBreaker onBreak ."); // 打开断路器
               },
               onReset: () =>
               {
                   Console.WriteLine("circuitBreaker onReset "); // 关闭断路器
               },
               onHalfOpen: () =>
               {
                   Console.WriteLine("circuitBreaker onHalfOpen"); // 半开
               }
            );

            // 定义重试策略
            var retry = Policy.Handle<HttpRequestException>().RetryAsync(3);
            // 定义降级策略
            var fallbackPolicy = Policy<string>.Handle<HttpRequestException>().Or<BrokenCircuitException>()
                .FallbackAsync("FALLBACK", (x) =>
                {
                    Console.WriteLine($"进行了服务降级 -- {x.Exception.Message}");
                    return Task.CompletedTask;
                })
                .WrapAsync(circuitBreaker.WrapAsync(retry));
            string memberJsonResult = "";

            do
            {
                memberJsonResult = await fallbackPolicy.ExecuteAsync(async () =>
                {
                    using var httpClient = new HttpClient();
                    httpClient.BaseAddress = new Uri($"http://localhost:5000");
                    var result = await httpClient.GetAsync("/api/user/" + 1);
                    result.EnsureSuccessStatusCode();
                    var json = await result.Content.ReadAsStringAsync();
                    return json;
                });
                Thread.Sleep(1000);
            } while (memberJsonResult == "FALLBACK");

            if (memberJsonResult != "FALLBACK")
            {
                var member = JsonConvert.DeserializeObject<User>(memberJsonResult);
                Console.WriteLine($"{member!.Id}---{member.Name}");
            }
```

当请求出错时-->重试-->两次异常-->打开断路器--->10S后-->半开断路器，再次尝试请求，如果正常则关闭断路器，反之，打开。

### 3. 微服务对接Polly

使用AutoFac AOP注入方式，实现Polly调用

（1）Nuget包引入

	- Autofac
	- Autofac.Extras.DynamicProxy
	- Autofac.Extensions.DependencyInjection

（2）创建Polly策略特性配置类，用于设计策略参数

```C#

    /// <summary>
    /// Polly策略特性配置类（用于设计策略参数）
    /// </summary>
    public class PollyPolicyConfigAttribute : Attribute
    {
        /// <summary>
        /// 最多重试几次，如果为0则不重试
        /// </summary>
        public int MaxRetryTimes { get; set; } = 0;

        /// <summary>
        /// 重试间隔的毫秒数
        /// </summary>
        public int RetryIntervalMilliseconds { get; set; } = 100;

        /// <summary>
        /// 是否启用熔断
        /// </summary>
        public bool IsEnableCircuitBreaker { get; set; } = false;

        /// <summary>
        /// 熔断前出现允许错误几次
        /// </summary>
        public int ExceptionsAllowedBeforeBreaking { get; set; } = 3;

        /// <summary>
        /// 熔断多长时间（毫秒）
        /// </summary>
        public int MillisecondsOfBreak { get; set; } = 1000;

        /// <summary>
        /// 执行超过多少毫秒则认为超时（0表示不检测超时）
        /// </summary>
        public int TimeOutMilliseconds { get; set; } = 0;

        /// <summary>
        /// 缓存多少毫秒（0表示不缓存），用“类名+方法名+所有参数ToString拼接”做缓存Key
        /// </summary>

        public int CacheTTLMilliseconds { get; set; } = 0;

        /// <summary>
        /// 回退方法
        /// </summary>
        public string? FallBackMethod { get; set; }
    }
```

（3）创建PollyPolicyAttribute属性（即拦截器），用于AOP注入

```C#

    /// <summary>
    /// 定义AOP特性类及封装Polly策略
    /// </summary>
    [AttributeUsage(AttributeTargets.Method)]
    public class PollyPolicyAttribute : Attribute, IInterceptor
    {
        private static ConcurrentDictionary<MethodInfo, AsyncPolicy> policies
            = new ConcurrentDictionary<MethodInfo, AsyncPolicy>();

        //private static readonly IMemoryCache memoryCache
        //    = new MemoryCache(new MemoryCacheOptions());

        public void Intercept(IInvocation invocation)
        {
            if (!invocation.Method.IsDefined(typeof(PollyPolicyConfigAttribute), true))
            {
                // 直接调用方法本身
                invocation.Proceed();
            }
            else
            {
                PollyPolicyConfigAttribute pollyPolicyConfigAttribute = invocation.Method.GetCustomAttribute<PollyPolicyConfigAttribute>()!;
                //一个PollyPolicyAttribute中保持一个policy对象即可
                //其实主要是CircuitBreaker要求对于同一段代码要共享一个policy对象
                //根据反射原理，同一个方法的MethodInfo是同一个对象，但是对象上取出来的PollyPolicyAttribute
                //每次获取的都是不同的对象，因此以MethodInfo为Key保存到policies中，确保一个方法对应一个policy实例
                policies.TryGetValue(invocation.Method, out AsyncPolicy? policy);
                //把本地调用的AspectContext传递给Polly，主要给FallbackAsync中使用
                // 创建Polly上下文对象(字典)
                Context pollyCtx= new Context();
                pollyCtx["invocation"] = invocation;

                lock (policies)//因为Invoke可能是并发调用，因此要确保policies赋值的线程安全
                {
                    if (policy == null)
                    {
                        policy = Policy.NoOpAsync();//创建一个空的Policy
                        if (pollyPolicyConfigAttribute.IsEnableCircuitBreaker)
                        {
                            policy = policy.WrapAsync(Policy.Handle<Exception>()
                                .CircuitBreakerAsync(pollyPolicyConfigAttribute.ExceptionsAllowedBeforeBreaking,
                                TimeSpan.FromMilliseconds(pollyPolicyConfigAttribute.MillisecondsOfBreak),
                                onBreak: (ex, ts) =>
                                {
                                    Console.WriteLine($"熔断器打开 熔断{pollyPolicyConfigAttribute.MillisecondsOfBreak / 1000}s.");
                                },
                                onReset: () =>
                                {
                                    Console.WriteLine("熔断器关闭，流量正常通行");
                                },
                                onHalfOpen: () =>
                                {
                                    Console.WriteLine("熔断时间到，熔断器半开，放开部分流量进入");
                                }));
                        }
                        if (pollyPolicyConfigAttribute.TimeOutMilliseconds > 0)
                        {
                            policy = policy.WrapAsync(Policy.TimeoutAsync(() =>
                                TimeSpan.FromMilliseconds(pollyPolicyConfigAttribute.TimeOutMilliseconds),
                                Polly.Timeout.TimeoutStrategy.Pessimistic));
                        }
                        if (pollyPolicyConfigAttribute.MaxRetryTimes > 0)
                        {
                            policy = policy.WrapAsync(Policy.Handle<Exception>()
                                .WaitAndRetryAsync(pollyPolicyConfigAttribute.MaxRetryTimes, i =>
                                TimeSpan.FromMilliseconds(pollyPolicyConfigAttribute.RetryIntervalMilliseconds)));
                        }
                        // 定义降级测试
                        var policyFallBack = Policy.Handle<Exception>().FallbackAsync((fallbackContent, token) =>
                        {
                            // 必须从Polly的Context种获取IInvocation对象
                            IInvocation iv = (IInvocation)fallbackContent["invocation"];
                            var fallBackMethod = iv.TargetType.GetMethod(pollyPolicyConfigAttribute.FallBackMethod!);
                            var fallBackResult = fallBackMethod!.Invoke(iv.InvocationTarget, iv.Arguments);
                            iv.ReturnValue = fallBackResult;
                            return Task.CompletedTask;
                        }, (ex, t) =>
                        {
                            Console.WriteLine("====================>触发服务降级");
                            return Task.CompletedTask;
                        });

                        policy = policyFallBack.WrapAsync(policy);
                        //放入到缓存
                        policies.TryAdd(invocation.Method, policy);
                    }
                }

                // 是否启用缓存
                if (pollyPolicyConfigAttribute.CacheTTLMilliseconds > 0)
                {
                    //用类名+方法名+参数的下划线连接起来作为缓存key
                    string cacheKey = "PollyMethodCacheManager_Key_" + invocation.Method.DeclaringType
                                                                       + "." + invocation.Method + string.Join("_", invocation.Arguments);
                    //尝试去缓存中获取。如果找到了，则直接用缓存中的值做返回值
                    //if (memoryCache.TryGetValue(cacheKey, out var cacheValue))
                    //{
                    //    invocation.ReturnValue = cacheValue;
                    //}
                    //else
                    {
                        //如果缓存中没有，则执行实际被拦截的方法
                        Task task = policy.ExecuteAsync(
                            async (context) =>
                            {
                                invocation.Proceed();
                                await Task.CompletedTask;
                            },
                            pollyCtx
                        );
                        task.Wait();

                        ////存入缓存中
                        //using var cacheEntry = memoryCache.CreateEntry(cacheKey);
                        //{
                        //    cacheEntry.Value = invocation.ReturnValue;
                        //    cacheEntry.AbsoluteExpiration = DateTime.Now + TimeSpan.FromMilliseconds(pollyPolicyConfigAttribute.CacheTTLMilliseconds);
                        //}
                    }
                }
                else//如果没有启用缓存，就直接执行业务方法
                {
                    Task task = policy.ExecuteAsync(
                            async (context) =>
                            {
                                invocation.Proceed();
                                await Task.CompletedTask;
                            },
                            pollyCtx
                        );
                    task.Wait();
                }
            }
        }

    }
```

（4）AutoFac IOC注册(Program.cs)

```C#
        public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.UseStartup<Startup>();
                })
        #region IOC容器
                .UseServiceProviderFactory(new AutofacServiceProviderFactory())
            .ConfigureContainer<ContainerBuilder>((context, buider) =>
                {
                    // 必须使用单例注册
                    buider.RegisterType<UserService>()
                    .As<IUserService>().SingleInstance().EnableInterfaceInterceptors();
                    buider.RegisterType<PollyPolicyAttribute>();

                });
        #endregion
```

(5)在服务接口上添加[Intercept(typeof(Class))]标签

```C#
    [Intercept(typeof(PollyPolicyAttribute))]//表示要polly生效
    public interface IUserService
    {
        User FindUser(int id);

        IEnumerable<User> UserAll();

        #region Polly
        [PollyPolicy]
        [PollyPolicyConfig(FallBackMethod = "UserServiceFallback",
            IsEnableCircuitBreaker = true,
            ExceptionsAllowedBeforeBreaking = 3,
            MillisecondsOfBreak = 1000 * 5,
            CacheTTLMilliseconds = 1000 * 20)]
        User AOPGetById(int id);

        Task<User> GetById(int id);
        #endregion

    }
```

