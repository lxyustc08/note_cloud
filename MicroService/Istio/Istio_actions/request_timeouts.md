- [Request Timeouts](#request-timeouts)
  - [Request timeouts](#request-timeouts-1)
  - [Understanding what happened](#understanding-what-happened)

# Request Timeouts

本实验将展示如何通过Istio设置envoy中的request timeouts。

实验开始之前需要做如下准备：

+ 安装Istio
+ 部署Bookinfo示例程序
+ 初始化Bookinfo的virtual-service-all-v1流量配置文件

## Request timeouts

1. 将发送至reviews的请求，引导至reviews:v2版本，该版本调用raings服务
   
   ```
   kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: reviews
   spec:
     hosts:
       - reviews
     http:
     - route:
       - destination:
           host: reviews
           subset: v2
   EOF
   ```

2. 在ratings service中引入2s延迟
   
   ```
   kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: ratings
   spec:
     hosts:
     - ratings
     http:
     - fault:
         delay:
           percent: 100
           fixedDelay: 2s
       route:
       - destination:
           host: ratings
           subset: v1
   EOF
   ```

3. 使用浏览器刷新页面，此时页面延迟2s后才刷新

4. 在reviews service中添加0.5s延时
   
   ```
   kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: reviews
   spec:
     hosts:
     - reviews
     http:
     - route:
       - destination:
           host: reviews
           subset: v2
       timeout: 0.5s
   EOF
   ```

5. 刷新页面，页面显示reviews服务不可用
   
   ![Alt Text](../Istio_picture/request_timesout_1.png)

上述步骤设置了0.5s超时，但页面刷新时间仍然花费了1s，原因在于在productpage service中引入了硬编码retry（hard-code retry，hard-code指不改变程序编码无法修改改变）

## Understanding what happened

In this task, you used Istio to set the request timeout for calls to the reviews microservice to half a second. By default the request timeout is disabled. Since the reviews service subsequently calls the ratings service when handling requests, you used Istio to inject a 2 second delay in calls to ratings to cause the reviews service to take longer than half a second to complete and consequently you could see the timeout in action.

You observed that instead of displaying reviews, the Bookinfo product page (which calls the reviews service to populate the page) displayed the message: Sorry, product reviews are currently unavailable for this book. This was the result of it receiving the timeout error from the reviews service.

If you examine the fault injection task, you’ll find out that the productpage microservice also has its own application-level timeout (3 seconds) for calls to the reviews microservice. Notice that in this task you used an Istio route rule to set the timeout to half a second. Had you instead set the timeout to something greater than 3 seconds (such as 4 seconds) the timeout would have had no effect since the more restrictive of the two takes precedence. More details can be found here.

One more thing to note about timeouts in Istio is that in addition to overriding them in route rules, as you did in this task, they can also be overridden on a per-request basis if the application adds an x-envoy-upstream-rq-timeout-ms header on outbound requests. In the header, the timeout is specified in milliseconds instead of seconds.
