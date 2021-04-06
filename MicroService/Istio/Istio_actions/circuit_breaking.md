- [Circuit Breaking](#circuit-breaking)
  - [Before Testing](#before-testing)
  - [Configuring the circuit breaker](#configuring-the-circuit-breaker)
  - [Adding a client](#adding-a-client)
  - [Tripping the circuit breaker](#tripping-the-circuit-breaker)

# Circuit Breaking

本部分将展示如何为连接、请求以及离群检测配置circuit breaking。

对于创建可靠的微服务应用而言，circuit breaking一种重要的模式，通过使用circuit breaking，用户可以对应用故障的潜在影响，延迟峰值以及其他由于网络非常态引起的不良效果进行控制。

本部分将展示如何配置circuit breaking规则并通过特意触发circuit breaker对配置进行测试。

## Before Testing

测试之前需要做的准备工作

+ 配置Istio
+ 启动httpbin示例
  
  配置了自动的istio注入

  ```
  kubectl apply -f samples/httpbin/httpbin.yaml
  ```

  未配置自动istio注入

  ```
  kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
  ```

## Configuring the circuit breaker

1. 向httpbin service创建一个应用了circuit breaker设置的destination rule
   
   ```
   kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: httpbin
   spec:
     host: httpbin
     trafficPolicy:
       connectionPool:
         tcp:
           maxConnections: 1
         http:
           http1MaxPendingRequests: 1
           maxRequestsPerConnection: 1
       outlierDetection:
         consecutive5xxErrors: 1
         interval: 1s
         baseEjectionTime: 3m
         maxEjectionPercent: 100
   EOF  
   ```

2. 验证destination rule生效
   
   ```
   kubectl get destinationrules.networking.istio.io  httpbin -o yaml
   ......
   spec:
     host: httpbin
     trafficPolicy:
       connectionPool:
         http:
           http1MaxPendingRequests: 1
           maxRequestsPerConnection: 1
         tcp:
           maxConnections: 1
       outlierDetection:
         baseEjectionTime: 3m
         consecutive5xxErrors: 1
         interval: 1s
         maxEjectionPercent: 100
   ```

## Adding a client

创建一个向httpbin service发送流量请求的client，该客户端是一个简单的流量测试客户端，被称为fortio，该客户端允许用户控制连接数量，并发量以及出HTTP调用的延迟，通过该客户端出发在Destination rules中配置的circuit breaker。

1. 使客户端与Istio sidecar proxy进行交互，在开启Istio自动注入配置下，执行如下命令
   
   ```
   kubectl apply -f samples/httpbin/sample-client/fortio-deploy.yaml
   ```

   在未开启Istio自动注入配置下，执行如下命令

   ```
   kubectl apply -f <(istioctl kube-inject -f samples/httpbin/sample-client/fortio-deploy.yaml)
   ```

2. 登入client pods，并使用fortio工具调用httpbin，使用fortio工具时，传入curl参数，表明仅进行1次调用
   
   ```
   export FORTIO_POD=$(kubectl get pods -lapp=fortio -o 'jsonpath={.items[0].metadata.name}')
   kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl -quiet http://httpbin:8000/get

   HTTP/1.1 200 OK
   server: envoy
   date: Tue, 06 Apr 2021 03:18:37 GMT
   content-type: application/json
   content-length: 622
   access-control-allow-origin: *
   access-control-allow-credentials: true
   x-envoy-upstream-service-time: 15

   {
     "args": {},
     "headers": {
       "Content-Length": "0",
       "Host": "httpbin:8000",
       "User-Agent": "fortio.org/fortio-1.11.3",
       "X-B3-Parentspanid": "b4deb036e18e4356",
       "X-B3-Sampled": "0",
       "X-B3-Spanid": "76a048edd5d92f7a",
       "X-B3-Traceid": "76811de5db4e4f09b4deb036e18e4356",
       "X-Envoy-Attempt-Count": "1",
       "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/httpbin;Hash=674d5b136f39b7c3faa33ed3145656df4498e05b5daf83c74e8c02fee954b9e9;Subject=\"\";URI=spiffe://cluster.local/ns/default/sa/default"
     },
     "origin": "127.0.0.1",
     "url": "http://httpbin:8000/get"
   }
   ```

   输出上述结果表明，forio工具部署成功。

## Tripping the circuit breaker

在上文中的Destination rule中配置了circuit breaker，具体配置为`maxConnections: 1`，`http1MaxPendingRequests: 1`。上述配置表明，当用户并发执行超过1个连接与1个请求时，用户将发现由于istio-proxy开启circuit导致的一些错误。

1. 使用2并发连接（-c 2）并发送20个请求（-n 20）
   
   ```
   kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 2 -qps 0 -n 20 -loglevel Warning http://httpbin:8000/get

   05:25:34 I logger.go:127> Log level is now 3 Warning (was 2 Info)
   Fortio 1.11.3 running at 0 queries per second, 4->4 procs, for 20 calls: http://httpbin:8000/get
   Starting at max qps with 2 thread(s) [gomax 4] for exactly 20 calls (10 per thread + 0)
   05:25:34 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   05:25:34 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   05:25:34 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   05:25:34 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   05:25:34 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   05:25:34 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   Ended after 93.664105ms : 20 calls. qps=213.53
   Aggregated Function Time : count 20 avg 0.0081530732 +/- 0.01059 min 0.000441187 max 0.047602844 sum 0.163061463
   # range, mid point, percentile, count
   >= 0.000441187 <= 0.001 , 0.000720594 , 20.00, 4
   > 0.004 <= 0.005 , 0.0045 , 55.00, 7
   > 0.005 <= 0.006 , 0.0055 , 80.00, 5
   > 0.011 <= 0.012 , 0.0115 , 85.00, 1
   > 0.016 <= 0.018 , 0.017 , 90.00, 1
   > 0.02 <= 0.025 , 0.0225 , 95.00, 1
   > 0.045 <= 0.0476028 , 0.0463014 , 100.00, 1
   # target 50% 0.00485714
   # target 75% 0.0058
   # target 90% 0.018
   # target 99% 0.0470823
   # target 99.9% 0.0475508
   Sockets used: 8 (for perfect keepalive, would be 2)
   Jitter: false
   Code 200 : 14 (70.0 %)
   Code 503 : 6 (30.0 %)
   Response Header Sizes : count 20 avg 161.05 +/- 105.4 min 0 max 231 sum 3221
   Response Body/Total Sizes : count 20 avg 668.75 +/- 280 min 241 max 853 sum 13375
   All done 20 calls (plus 0 warmup) 8.153 ms avg, 213.5 qps
   ```

   注意结果，该结果表明`istio-proxy`留有了部分余地

   ```
   Code 200 : 14 (70.0 %)
   Code 503 : 6 (30.0 %)
   ```

2. 将并发连接数提升至3（-c 3），请求数设置为30
   
   ```
   kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 3 -qps 0 -n 30 -loglevel Warning http://httpbin:8000/get

   06:52:04 I logger.go:127> Log level is now 3 Warning (was 2 Info)
   Fortio 1.11.3 running at 0 queries per second, 4->4 procs, for 30 calls: http://httpbin:8000/get
   Starting at max qps with 3 thread(s) [gomax 4] for exactly 30 calls (10 per thread + 0)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   06:52:04 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
   Ended after 33.323099ms : 30 calls. qps=900.28
   Aggregated Function Time : count 30 avg 0.0029133326 +/- 0.002855 min 0.000526769 max 0.009059439 sum 0.087399979
   # range, mid point, percentile, count
   >= 0.000526769 <= 0.001 , 0.000763384 , 46.67, 14
   > 0.001 <= 0.002 , 0.0015 , 66.67, 6
   > 0.005 <= 0.006 , 0.0055 , 80.00, 4
   > 0.006 <= 0.007 , 0.0065 , 83.33, 1
   > 0.007 <= 0.008 , 0.0075 , 96.67, 4
   > 0.009 <= 0.00905944 , 0.00902972 , 100.00, 1
   # target 50% 0.00116667
   # target 75% 0.005625
   # target 90% 0.0075
   # target 99% 0.00904161
   # target 99.9% 0.00905766
   Sockets used: 22 (for perfect keepalive, would be 3)
   Jitter: false
   Code 200 : 10 (33.3 %)
   Code 503 : 20 (66.7 %)
   Response Header Sizes : count 30 avg 76.666667 +/- 108.4 min 0 max 230 sum 2300
   Response Body/Total Sizes : count 30 avg 444.66667 +/- 288 min 241 max 852 sum 13340
   All done 30 calls (plus 0 warmup) 2.913 ms avg, 900.3 qps
   ```

   注意上述结果中的请求状态结果，该结果显示，仅有33.3%的请求成功了，剩余请求被circuit breaking中断处理了。

   ```
   Code 200 : 10 (33.3 %)
   Code 503 : 20 (66.7 %)
   ```

3. 查看此时的`istio-proxy`状态：
   
   ```
   kubectl exec "$FORTIO_POD" -c istio-proxy -- pilot-agent request GET stats | grep httpbin | grep pending
   cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.default.rq_pending_open: 0
   cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.high.rq_pending_open: 0
   cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_active: 0
   cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_failure_eject: 0
   cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_overflow: 26
   cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_total: 28
   ```

   注意值`upstream_rq_pending_overflow: 26`，该值与1 2次实验503状态码数量和一致，该值表明总共有26个请求被标记按照断路器配置处理项处理。