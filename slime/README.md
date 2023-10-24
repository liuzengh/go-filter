English | [中文](README.zh_CN.md)

## Supported Protocols

[![Go Reference](https://pkg.go.dev/badge/trpc.group/trpc-go/trpc-filter/slime.svg)](https://pkg.go.dev/trpc.group/trpc-go/trpc-filter/slime)
[![Go Report Card](https://goreportcard.com/badge/trpc.group/trpc-go/trpc-filter/slime)](https://goreportcard.com/report/trpc.group/trpc-go/trpc-filter/slime)
[![Tests](https://github.com/trpc-ecosystem/go-filter/actions/workflows/slime.yml/badge.svg)](https://github.com/trpc-ecosystem/go-filter/actions/workflows/slime.yml)
[![Coverage](https://codecov.io/gh/trpc-ecosystem/go-filter/branch/main/graph/badge.svg?flag=slime&precision=2)](https://app.codecov.io/gh/trpc-ecosystem/go-filter/tree/main/slime)


<span style="color:red">**DO NOT enable retry/hedging for non-idempotent requests**</span>.  
<span style="color:red">Not all protocols can use retry/hedging</span>.  
Yaml config in Chapter Slime may not be suitable for non tRPC protocols. In this case, you could use the
basic package in sesion "Introduce to Basic Retry/Hedging Packages" directly.

| protocols | retry | hedging | note |
|:---:|:---:|:---:|:---|
|tRPC|✓|✓| Native tRPC protocol. |
|trpc SendOnly|✗|✗| Does not support, retry/hedging depends on the return error, however, SendOnly request has no response. |
|trpc Stream|✗|✗| Does not support. |
|[http](https://github.com/trpc-group/trpc-go/tree/main/http)|✓|✓||
|[Kafaka](https://github.com/trpc-ecosystem/go-database/tree/main/kafka)|✓|✗| Hedging is not supported. |
|[MySQL](https://github.com/trpc-ecosystem/go-database/tree/main/mysql)|★|★| <span style="color:red"> support all method except [Query](https://github.com/trpc-ecosystem/go-database/blob/6f75e87fecfc5411e54d93fd1aad5e7afa9a0fcf/mysql/client.go#L40) and [Transaction](https://github.com/trpc-ecosystem/go-database/blob/6f75e87fecfc5411e54d93fd1aad5e7afa9a0fcf/mysql/client.go#L42)</span>. These two methods use lambda as parameters, and slime cannot guarantee concurrency safety. You can use `slime.WithDisabled` to disable retry/hedging. |


## Background
Retry is a very simple idea. When the original request fails, a retry request is initiated. In a narrow sense, retry is a conservative strategy, and a new request will only be triggered when the last request fails. Users who prefer less response time may wish to use a more aggressive strategy, **hedging**. Jeffrey Dean first mentioned hedging in [the tail at scale](https://cacm.acm.org/magazines/2013/2/160173-the-tail-at-scale/pdf) to solve the impact of long-tail requests when the number of fan-outs is large.

Simply put, the hedging strategy is not passively waiting for the last request to time out or fail. A new request will be triggered if a successful replay packet is not received within the hedging delay time(less than the request timeout). Unlike the retry strategy, there may be multiple in-flight requests at the same time. The first successful request will be handed over to the application layer, and the responses of other requests will be ignored.

Note that these two strategies are mutually exclusive, and users can only choose one of them.

The implementation of the retry strategy is relatively simple. There are also some implementations of the hedging
strategy in industry:
* [gRPC](https://github.com/grpc/grpc-java): [A6-client-retries.md](https://github.com/grpc/proposal/blob/master/A6-client-retries.md)
gives a very detailed introduction of gRPC design. gRPC-java has implemented it.
* [bRPC](https://github.com/apache/incubator-brpc): In bRPC, a hedging request is called a backup request. This [doc](https://github.com/apache/incubator-brpc/blob/master/docs/cn/backup_request.md) gives a brief introduction, and its c++ implementation is relative simple.
* [finagle](https://github.com/twitter/finagle): finagle is a java RPC open source framework, it also implements [backup request](https://twitter.github.io/finagle/guide/MethodBuilder.html#backup-requests).
* [pegasus](https://github.com/apache/incubator-pegasus): Pegasus is a kv database that supports simultaneous reading data from multiple copies. [Backup request] is used to improve performance.
* [envoy](https://www.envoyproxy.io/docs/envoy/latest/): Envoy, as a proxy service, is widely used in cloud native. It also supports [request hedging](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_routing#request-hedging).

This article will introduce retry/hedging of the tRPC framework. In next section, we briefly introduced the rationale for retry hedging. The next two sectinos describe more implementation details. We introduced the basic package of retry/hedging, and the slime is a manager based on these basic capabilities, which provides you with yaml-based configuration. Finally, we list some problems you may encounter.

## Principles
In this chapter, we will show the basic principles of hedging and retrying through examples, and briefly introduce some other capabilities that you may need to pay attention to.

### Retry Strategy
As the name suggests, retry on error.

![ 'image.png'](./.docs/retry.png)

In the figure above, the client has tried three times: orange, blue, and green. The first two failed, and a random backoff was made before each attempt to prevent request glitches. Eventually the third attempt succeeded and returned to the application layer. For each attempt, we try to send it to a different node.

Generally, a retry policy requires the following configurations:
- Maximum number of retries: Once exhausted, the last error is returned.
- Backoff time: The actual backoff time is random(0, delay).
- Retryable error code: If the returned error is not retryable, stop retrying immediately and return the error to the application layer.

### Hedging Strategy
As we introduced in the background, hedging can be seen as a more aggressive and complex type of retry.

![ 'image.png'](./.docs/hedging.png)

In the picture above, the client has tried 4 times in total: orange, blue, green, and purple.
- <span style="color:F2B442">Orange</span> is the first attempt. After being initiated by the client, server2 received it soon. However, due to network and other problems of server2, its correct response packet was not late until the green request was successful and returned to the application layer. Although it was successful, we have to discard it because we have already sent another success response back to the application layer.
- <span style="color:2765FF">Blue</span> is the second attempt. We initiated a new attempt since the orange request has not been returned after the hedging delay. This time, server1 was chosen for this attempt (we will try to choose a different node for each attempt as much as possible). The response of the blue attempt is faster and returns before the hedging delay. But it failed. We **immediately** start a new attempt.
<span style="color:A4D955">Green</span> is the third attempt. Although its response may be a bit slow (beyond the hedging delay, thus triggering a new attempt), but it worked! As soon as we receive the first successful response, we immediately return it to the application layer.
<span style="color:B937F6">Purple</span> is the fourth attempt. As soon as it was initiated, we received a success response of green. For purple request, it may be in many states: the request is still in the client tRPC, at this time, we have the opportunity to cancel it; the request has entered the client's kernel or has been sent by the network card, in any case, we have no chance to cancel it. <span style="color:F20F79">✘</span> on a purple request indicates that we will suppress purple requests whenever possible. Note that even though the purple request eventually makes it to server2 successfully, its response is dropped like orange.

As you can see, hedging is more like a **concurrent** retry with a **waiting time**. Hedging has no back-off mechanism, once it receives an error response, it will immediately initiate a new attempt. In general, we recommend using hedging strategies only when you need to address the long tail problem. For ordinary error retry, please use a simpler and clearer retry mechanism.

Generally, hedging will have the following configuration:
- Maximum number of retries: Once exhausted, wait for and return the last response, regardless of whether it succeeded or failed.
- Hedging delay: If no response is received within the hedging delay, a new attempt will be initiated immediately.
- Non-fatal errors: Returning a fatal error will abort hedging immediately, waiting for and returning the last response, regardless of whether it succeeded or failed. Returning a non-fatal error will immediately trigger a new attempt (the hedging delay timer will be reset).

### the Order of Interceptor
In tRPC-Go, the hedging/retrying is implemented in interceptors.

When a request passes through the retry/hedging interceptor, it may generate multiple sub-requests, and each sub-request executes subsequent interceptors.  
For monitoring interceptors, you must pay attention to their relative position to retry/hedging interceptors. If they are located before the retry/hedging, they will only be counted once for each request of the application layer; if they are located after the retry/hedging, then they will be counted for each retrying hedging request.

When you use a retry/hedging interceptor, be sure to give some thought to how it relates to other interceptors.

### Server Pushback
Server pushback is used by the server to explicitly control the client's retry/hedging strategy.  
When the load on the server is relatively high, and you want the client to reduce the retry/hedging frequency, you can specify a delay time T in the return packet, and the client will delay the next retry/hedging sub-request for T time. This function is more commonly used by the server to instruct the client to stop retrying/hedging, by setting delay to `-1`.

In general, you shouldn't care whether you need to set server pushback or not. In subsequent planning, the framework will automatically determine how to set server pushback according to the current load of the service.

### Load Balance
Because retry/hedging is implemented as an interceptor, and load balancing occurs after the interceptor, each
sub-request will trigger a load balancing.

!['image.png'](./.docs/loadbalance.png)

For hedging requests, you may want each sub-request to be sent to a different node. We implemented a mechanism that allows multiple sub-requests to communicate to get nodes that other sub-requests have already visited. A load balancer can take advantage of this mechanism and only return unvisited nodes. Of course, this requires the cooperation of the load balancer. Currently, there are only two built-in random load balancing strategies in the framework to support it.  
Don't get discouraged if you're using a load balancer that doesn't support skipping already visited nodes. Under normal circumstances, the round-robin or random load balancer itself realizes that sub-requests are sent to different nodes in a sense, even if they are sent to the same node occasionally, there will be no major problems. For a special hash-type load balancer (routing to a specific node according to a specific key, rather than a class of nodes), it may not support this function at all. In fact, using hedging on this type of load balancer Strategies are pointless.

## Introduce to Basic Retry/Hedging Packages
This chapter only briefly introduces the basic package of retry/hedging as the basis of next section. Although we provide some usage examples, please <span style="color:red">try to avoid using them directly at the application layer</span>. You should use slime to enable retry/hedging.

### [retry](./retry)
[retry](./retry) provides the basic retry strategy.

`New` creates a new retry strategy, you must specify the maximum number of retries and retryable error codes. You can also customize the retryable error through `WithRetryableErr`, which has an OR relationship with the retryable error codes.

Retry provides two default backoff strategies: `WithExpBackoff` and `WithLinearBackoff`. You can also customize the backoff strategy via `WithBackoff`. At least one of these three backoff strategies needs to
be provided. If you provide more than one, their priority is:  
`WithBackoff` > `WithExpBackoff` > `WithLinearBackoff`

You may be wondering why `WithSkipVisitedNodes(skip bool)` has an extra `skip` boolean variable? In fact, we distinguish three situations here:
1. The user does not explicitly specify whether to skip already visited nodes;
2. The user explicitly specifies to skip already visited nodes;
3. The user explicitly specifies not to skip already visited nodes.

These three states have different impacts on load balancing.
For the first case, the load balancer should return unvisited nodes as much as possible. We allow it to return a visited node if all nodes have been visited. This is the default policy.  
For the second case, the load balancer must return unvisited nodes. If all nodes have already been visited, it should return no nodes available error.  
For the third case, the load balancer can return to any node at will.  

As described in previous section, `WithSkipVisitedNodes` requires the cooperation of load balancing. If the load balancer does not implement this function, no matter whether the user invokes this option or not, it finally corresponds to the third situation.

`WithThrottle` can specify a throttle for this strategy.

You can specify a retry policy for an RPC request in the following ways:
```Go
r, _ := retry.New(4, []int{errs.RetClientNetErr}, retry.WithLinearBackoff(time.Millisecond*5))
rsp, _ := clientProxy.Hello(ctx, req, client.WithFilter(r.Invoke))
```

### [Hedging](./hedging)
[Hedging](./hedging) provides the basic hedging strategy.

`New` creates a new hedging strategy. You must specify the maximum number of retries and non-fatal error codes. You can also customize non-fatal errors through `WithNonFatalError`, which has an OR relationship with non-fatal error codes.

The hedging package provides two ways to set the hedging delay. `WithStaticHedgingDelay` sets a static delay.
`WithDynamicHedgingDelay` allows you to register a function that returns a time each time it is called as the hedging delay. These two methods are mutually exclusive, and the latter overrides the former when specified multiple times.

`WithSkipVisitedNodes` behaves the same as retry, please refer to the previous section.

`WithThrottle` can specify a throttle for the hedging strategy.

You can specify a hedging strategy for an RPC request in the following ways:
```Go
h, _ := hedging.New(2, []int{errs.RetClientNetErr}, hedging.WithStaticHedgingDelay(time.Millisecond*5))
rsp, _ := clientProxy.Hello(ctx, req, client.WithFilter(h.Invoke))
```

### [Throttle](./throttle)
[Throttle](./throttle) is a used to avoid retry/hedging write amplification.

The `throttler` interface provides three methods:
```Go
type throttler interface {
	Allow() bool
	OnSuccess()
	OnFailure()
}
```
Every time a retry/hedging sub-request is sent (excluding the first request), `Allow` will be called. If `false` is returned, all subsequent sub-requests requested by the application layer will not be executed again, which is regarded as "maximum The number of hedges has been exhausted".

Whenever a retry/hedge sub-request response is received, `OnSuccess` or `OnFailure` will be called as appropriate.

Hedging/Retrying will generate write amplification, while rate limiting is to avoid service avalanche caused by retrying/hedging. When you initialize a `throt` like below, and bind it to a `Hello` RPC,
```Go
throt, _ := throttle.NewTokenBucket(10, 0.1)
r, _ := retry.New(3, []int{errs.RetClientNetErr}, retry.WithLinearBackoff(time.Millisecond*5))
tr := r.NewThrottledRetry(throt)
rsp, _ := clientProxy.Hello(ctx, req, client.WithFilter(tr.Invoke))
```
the total number of `Hello` requests due to retry/hedging will not exceed 110% of the number of application layers (each successful request will add 0.1 to the token, and each failed request will reduce the token by 1, which is equivalent to 10 Only one successful request can be exchanged for a retry/hedging opportunity), and the number of retry/hedging requests (continuous failure) will not be greater than 5 (5 = 10 / 2, only when the number of tokens is greater than half, `Allow ` will return `true`).

### about Timeout Error
In tRPC-Go, [`RetClientTimeout`](https://github.com/trpc-group/trpc-go/blob/71941c0f7e32cec11d48c0d6b5c28122788f57e8/errs/errs.go#L47), namely 101 error,
corresponds to the application layer time out. Retry/hedging follows this mechanism and returns an error as soon as `ctx` times out. Therefore, <span style="color:red">it doesn't make sense to use 101 as a retry/hedge error code</span>. In this case, we recommend that you use the hedging function and configure a reasonable hedging delay (the hedging delay is your expected timeout). Note that the hedging delay should be less than the application layer timeout.

## Slime

Slime provides file configuration functions on top of the two basic packages retry and hedging. With slime, you can manage the retry/hedging strategy in the framework configuration. Like other tRPC-Go plugins, first import the slime package anonymously:
```go
import _ "trpc.group/trpc-go/trpc-filter/slime"
```
Then, config the following yaml:
```yaml
--- # retry/hedging strategy
retry1: &retry1 # this is a yaml reference syntax that allows different services to use the same retry strategy
  # use a random name if omitted.
  # if you need to customize backoff or retryable business errors, you must explicitly provide a name, which will be
  # used as the first parameter of the slime.SetXXX method.
  name: retry1
  # default as 2 when omitted.
  # no more than 5, truncate to 5 if exceeded.
  max_attempts: 4
  backoff: # must provide one of exponential or linear
    exponential:
      initial: 10ms
      maximum: 1s
      multiplier: 2
  # when omitted, the following four framework errors are retried by default:
  #  21: RetServerTimeout
  # 111: RetClientConnectFail
  # 131: RetClientRouteErr
  # 141: RetClientNetErr
  # for tRPC-Go framework error codes, please refer to: https://github.com/trpc-group/trpc-go/tree/main/errs
  retryable_error_codes: [ 141 ]

retry2: &retry2
  name: retry2
  max_attempts: 4
  backoff:
    linear: [100ms, 500ms]
  retryable_error_codes: [ 141 ]
  skip_visited_nodes: false # omit, false and true correspond to three different cases

hedging1: &hedging1
  # use a random name if omitted.
  # if you need to customize hedging_delay or non-fatal errors, you must explicitly provide a name, which will be used
  # as the first parameter of the slime.SetHedgingXXX method.
  name: hedging1
  # default as 2 when omitted.
  # no more than 5, truncate to 5 if exceeded.
  max_attempts: 4
  hedging_delay: 0.5s
  # when omitted, the following four errors default to non-fatal errors:
  # 21: RetServerTimeout
  # 111: RetClientConnectFail
  # 131: RetClientRouteErr
  # 141: RetClientNetErr
  non_fatal_error_codes: [ 141 ]

hedging2: &hedging2
  name: hedging2
  max_attempts: 4
  hedging_delay: 1s
  non_fatal_error_codes: [ 141 ]
  skip_visited_nodes: true # omit, false and true correspond to three different cases.

--- # client config
client: &client
  filter: [slime] # filter must cooperate with plugin, both are indispensable
  service:
    - name: trpc.app.server.Welcome
      retry_hedging_throttle: # all retry/hedging strategies under this service will be bound to this rate limit
        max_tokens: 100
        token_ratio: 0.5
      retry_hedging: # service uses policy retry1 by default
        retry: *retry1 # dereference retry1
      methods:
        - callee: Hello # use retry policy retry2 instead of retry1 of parent service
          retry_hedging:
            retry: *retry2
        - callee: Hi # use hedging policy hedging1 instead of retry1 of parent service
          retry_hedging:
            hedging: *hedging1
        - callee: Greet # empty retry_hedging means no retry/hedging policy
          retry_hedging: {}
        - callee: Yo # retry_hedging is missing, use retry1 of parent service by default
    - name: trpc.app.server.Greeting
      retry_hedging_throttle: {} # forcibly turn of rate limit
      retry_hedging: # service uses hedging2 by default
        hedging: *hedging2
    - name: trpc.app.server.Bye
      # missing rate limit, use the default one.
      # there's no retry/hedging policy at service level.
      methods:
        - callee: SeeYou # SeeYou use retry1 as its own retry policy
          retry_hedging:
            retry: *retry1

plugins:
  slime:
    # we reference the entire client here. Of course, you can configure client.service separately under default.
    default: *client
```

> The above configuration file uses an important feature in yaml, namely [reference](https://en.wikipedia.org/wiki/YAML#Advanced_components).
For duplicate nodes, you can reuse them by reference.

### Retry/Hedging Policy as an [Entity](https://en.wikipedia.org/wiki/Domain-driven_design#Building_blocks)

In the configuration above, we defined four retry/hedging policies and referenced them in `client`. Each strategy, in addition to the required parameters, has a new field `name`, which is used as a **unique** identifier for the entity.
In the previous chapters, we mentioned some options, such as `WithDynamicHedgingDelay`, they cannot be configured in the file and need to be used in the code, where `name` is the key to using these options in the code. In slime, we provide the following functions to set additional options.
```Go
func SetHedgingDynamicDelay(name string, dynamicDelay func() time.Duration) error
func SetHedgingNonFatalError(name string, nonFatalErr func(error) bool)
func SetRetryBackoff(name string, backoff func(attempt int) time.Duration) error
func SetRetryRetryableErr(name string, retryableErr func(error) bool) error
```

Note that for the `backoff` of the retry strategy, you can only choose between `exponential` and `linear`. If you provide both, we'll take `exponential` whichever.

### Unification with Framework Config

In the plugin configuration `plugins`, the plugin type must be `slime` and the plugin name must be `default`. slime will load all retry/hedging strategies into a plugin, namely default, according to the configuration file. default provides an interceptor, which automatically takes effect for
all services or methods configured with retry/hedging.

As you may have noticed, the `client` key is similar to the client framework configuration, except that it has some new keys, such as `retry_hedging`, `methods`, etc. We deliberately designed it this way, in order to be able to reuse the original framework configuration. If you plan to introduce slime into the existing client, then you only need to add some key values under the `client` key of the framework configuration.

Hedging is a more aggressive retry strategy. When configuring retry/hedging strategies, you can only choose one of them:
```yaml
retry_hedging:
  retry: *retry1
  # hedging: *hedging1  # do not config hedging if you have chosen retry
```
If you config both retry and hedging, then we will use hedging instead of retry.  
If you config `retry_hedging: {}`, then the strategy is equivalent to disable retry/hedging. Note that this is different from `retry_hedging:`, the former is configured with the key `retry_hedging`, but its content is empty, the latter is equivalent to no key `retry_hedging`.

You can specify a retry/hedging strategy for the entire service, just add the `retry_hedging` key under `service`, or you can refine it to a specific method by adding `callee` in `method`.  

In the configuration file, the service `trpc.app.server.Welcome` uses `retry1` as the retry strategy.  
`Hello` overrides service retry strategy `retry1` with retry strategy `retry2`.  
`Hi` overrides service retry strategy `retry1` with hedging strategy `hedging1`.  
`Greeter` overrides service policy `retry1` with **null policy**.  
`Yo` inherits the service policy `retry1`.  
Other methods that are not explicitly configured inherit the service policy `retry1` by default.  
All methods of the service `trpc.app.server.Greeting` use the hedging strategy `hedging2`.

### Throttle
In slime, rate limit is based on service.  
By default, slime enables the rate limit for each service, configured as `max_tokens: 10` and `token_ratio: 0.1`.  
You can also customize `max_tokens` and `token_ratio` like service `trpc.app.server.Welcome`.  
If you want to turn off throttle, you need to configure it like this: `retry_hedging_throttle: {}`.

### Interceptor
When the slime plugin is initialized, it will automatically register the slime interceptor.  
The interceptor `slime` must be added to `filter` to enable slime plugin.
```yaml
client:
  filter: [slime]
  service:
    - # you can also add interceptor inside the service
      #filter: [slime]
```
Slime will generate multiple sub-requests, please pay attention to its order with other interceptors.

### Skip the Visited Nodes
As we described in previous section, you can also specify in the configuration whether to skip nodes that have already sent a request.  
`retry1` and `hedging1` are not configured with `skip_visited_nodes`, they correspond to the first case. `retry2` explicitly specifies `skip_visited_nodes` to be `false`, which corresponds to the third case. `hedging2` explicitly specifies `skip_visited_nodes` to be `true`, which corresponds to the second case.

Note that this feature requires the cooperation of a load balancer. If the load balancer does not implement that, then it will correspond to the third case.

### Disable Retry/Hedging for a Single Request
Slime support to disable retry/hedge for a single request by creating a new context.  
This function usually cooperates with trpc-database to make the retry/hedging configuration take effect only for read requests (or idempotent requests), while skipping write requests. For example, for trpc-database/mysql:
```go
c := mysql.NewClientProxy(/* omitted args */)
err := c.QueryRow(trpc.BackgroundContext(), /* omitted args */) // retry/hedging is default enabled by config
_, err = c.Exec(slime.WithDisabled(trpc.BackgroundContext()), /* omitted args */) // disable retry/hedging for this request by ctx
```
Note that this function is only available for slime, and slime/retry or slime/hedging do not provide this function.

## Visualization
Slime provides two visualization capabilities, one is conditional log and the other is metrics.
### Conditional Log
Whether hedging or retrying, they have an option called `WithConditionalLog`.
[this](https://github.com/trpc-ecosystem/go-filter/blob/779875aa0e61af72a7fe5792c7833ceef110adad/slime/retry/retry.go#L237) is for retry, and
[this](https://github.com/trpc-ecosystem/go-filter/blob/779875aa0e61af72a7fe5792c7833ceef110adad/slime/hedging/hedging.go#L188) is for hedging, these two
([retry](https://github.com/trpc-ecosystem/go-filter/blob/779875aa0e61af72a7fe5792c7833ceef110adad/slime/opts.go#L247),
[hedging](https://github.com/trpc-ecosystem/go-filter/blob/779875aa0e61af72a7fe5792c7833ceef110adad/slime/opts.go#L96)) are for slime.

Conditional logging requires two parameters, one is `log.Logger`:
```go
type Logger interface {
	Println(string)
}
```
another is `func(stat view.Stat) bool`.

`view.Stat` in the condition function provides a status of the application layer request execution process. You can decide whether to output retry/hedge logs based on these data. For example, the following conditional function tells slime to only output the log when the first two retries fail and the third one succeeds after a total of three retries:
```go
var condition = func(stat view.Stat) bool {
	attempts := stat.Attempts()
	return len(attempts) == 3 &&
		attempts[0].Inflight() &&
		attempts[1].Inflight() &&
		!attempts[2].Inflight() &&
		attempts[2].Error() == nil
}
```

`Logger` only needs a simple `Println(string)` method. You can wrap one based on any log library. For example, the following is a console-based log:
```go
type ConsoleLog struct{}

func (l *ConsoleLog) Println(s string) {
	log.Println(s)
}
```
Here is a slime log on the console:
!['image.png'](./.docs/logs.png)
There are a few points you need to pay attention to:
* All slime logs requested by an application layer correspond to one `Println` in `log.Logger`, which is called lazy log in slime, as shown in the first line in the screenshot.
* Slime's logs are formatted with newlines, tabs, etc.
* The last slime log is a summary of all attempts.

### Metrics
Similar to conditional logs, retry/hedge monitoring is also based on
[`view.Stat`](./view/stat.go).

slime provides four metrics: application layer request number, actual request number, application layer time
consumption, and actual time consumption.  
All monitoring items have three tags: caller, callee, method.  
For the number of application layer requests and time consumption, they have the following additional tags: total number of attempts, error code of the final error, whether it is throttled, the number of outstanding requests (only if hedging can be non-zero), whether the backend prohibits retry/hedging.  
For the actual number of requests and the actual time-consuming, they have the following additional tags: error code, whether it is not completed, whether the backend explicitly prohibits retry/hedging.

#### Prometheus
Slime supports reporting metrics to Prometheus. Import dependencies:
```go
import prom "trpc.group/trpc-go/trpc-filter/slime/view/metrics/prometheus"
```
Use `prom.NewEmitter` to initialize an Emitter.  
How to use prometheus can refer to [Official Documentation](https://prometheus.io/docs/guides/go-application/).
