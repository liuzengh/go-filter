## tRPC-Go [hystrix] fuse protection plugin

[![Go Reference](https://pkg.go.dev/badge/trpc.group/trpc-go/trpc-filter/hystrix.svg)](https://pkg.go.dev/trpc.group/trpc-go/trpc-filter/hystrix)
[![Go Report Card](https://goreportcard.com/badge/trpc.group/trpc-go/trpc-filter/hystrix)](https://goreportcard.com/report/trpc.group/trpc-go/trpc-filter/hystrix)
[![Tests](https://github.com/trpc-ecosystem/go-filter/actions/workflows/hystrix.yml/badge.svg)](https://github.com/trpc-ecosystem/go-filter/actions/workflows/hystrix.yml)
[![Coverage](https://codecov.io/gh/trpc-ecosystem/go-filter/branch/main/graph/badge.svg?flag=hystrix&precision=2)](https://app.codecov.io/gh/trpc-ecosystem/go-filter/tree/main/hystrix)

### hystrix plugin introduction

The plugin is based on Netflix's open source hystrix component, see https://github.com/afex/hystrix-go

> Hystrix is a latency and fault tolerance library designed to isolate access points to remote systems, services, and third-party libraries, stop cascading failures, and achieve resilience in complex distributed systems where failures are inevitable.

### hystrix plugin instructions

1. Add import in main.go

   ```go
   import （
   	_ "trpc.group/trpc-go/trpc-filter/hystrix"
   ）
   ```

2. Add corresponding filter in TRPC framework configuration
   ```yaml
   # If you need to fuse the method of the server, configure the server-side filter.
   server:
     ...
     filter:
       ...
       - hystrix
   # If you need to fuse the method of the client, configure the client-side filter.
   client:
     ...
     filter:
       ...
       - hystrix
   ```

3. Add plugin configuration in TRPC framework configuration file
   ```yaml
   plugins:                                     # Plugin configuration
     circuitbreaker:
       hystrix:
         /trpc.qq_news.user_info.UserInfo/Api1: # Business routing【server】trpc.Message(ctx).ServerRPCName(); 【client】trpc.Message(ctx).ClientRPCName()
           timeout: 1000              # Overtime（ms）
           maxconcurrentrequests: 100 # The maximum number of concurrent requests.
           requestvolumethreshold: 30 # After more than ? requests, the fuse will be turned on according to the error ratio.
           sleepwindow: 2000          # Fuse time（ms）
           errorpercentthreshold: 10  # Turn on the error ratio of fusing.
         /trpc.qq_news.user_info.UserInfo/Api2: 
           timeout: 2000
           maxconcurrentrequests: 100
           requestvolumethreshold: 3
           sleepwindow: 5000
           errorpercentthreshold: 10
        "*": # Set up wildcards.
           timeout: 2000
           maxconcurrentrequests: 100
           requestvolumethreshold: 3
           sleepwindow: 5000
           errorpercentthreshold: 10
        _/trpc.qq_news.user_info.UserInfo/Api3: # Exclude an interface when global configuration is enabled.
        _/trpc.qq_news.user_info.UserInfo/Api4: # Exclude an interface when global configuration is enabled.
   ```
The ``hystrix`` component can configure different circuit breaking rules for different routes, as above, they are ``/trpc.qq_news.user_info.UserInfo/Api1``, ``/trpc.qq_news.user_info.UserInfo/Api2`` Different circuit breaking policies are configured. For Api1, the timeout period of this method is 1000ms, and the maximum number of concurrent requests is 100. When the number of requests exceeds 30, the fusing judgment strategy will be enabled. When the error rate reaches 10%, it will enter the fusing time of 2000ms, and return `` hystrix: circuit open`` error. At the same time, the configured business route can be either the server-side path or the dependent client-side path.


     The ``hystrix`` component can configure different fuse rules for different routes, ```"*"``` sets the general configuration, and the configuration prefixed with ```_``` is when the wild card is enabled Go down and delete some configurations. Priority: individual settings > globbing. ```"*"``` and ```_``` If these conflict with your routing, you can use ```hystrix.WildcardKey or hystrix.ExcludeKey ``` to modify these two variables.

4. hystrix monitoring data implementation guide
   * First implement metricCollector.
   ```go
   type testMetricCollector struct{
	attemptsPrefix          string
	errorsPrefix            string
	successesPrefix         string
	failuresPrefix          string
	rejectsPrefix           string
	shortCircuitsPrefix     string
	timeoutsPrefix          string
	fallbackSuccessesPrefix string
	fallbackFailuresPrefix  string
   }

   // Update ...
   func (m *testMetricCollector) Update(r metricCollector.MetricResult) {
        // Use the metric in trpc-go for data calculation.
   }

   // Reset ...
   func (m *testMetricCollector) Reset() {}

   func newTestMetircCollector(name string) metricCollector.MetricCollector {
    return &testMetricCollector{
		attemptsPrefix:          name + ".attempts",
		errorsPrefix:            name + ".errors",
		successesPrefix:         name + ".successes",
		failuresPrefix:          name + ".failures",
		rejectsPrefix:           name + ".rejects",
		shortCircuitsPrefix:     name + ".shortCircuits",
		timeoutsPrefix:          name + ".timeouts",
		fallbackSuccessesPrefix: name + ".fallbackSuccesses",
		fallbackFailuresPrefix:  name + ".fallbackFailures",
    }
   }
   ```

   * Register MetricCollector.
   ```go
	hystrix.RegisterCollector(newTestMetircCollector)
   ```
