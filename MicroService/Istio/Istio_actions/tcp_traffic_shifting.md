# TCP Traffic Shifting

本部分内容将对Istio的TCP流量迁移功能进行实验，本处实验首先将用户的100%流量发送至微服务tcp-echo:v1，然后利用Istio的weighted routing特性将20%的流量发送至微服务tcp-echo:v2

实验开始之前做如下准备：

+ 部署Istio
+ 查看Istio的Traffic management内容部分

## Set up test environment

1. 创建名称为istio-io-tcp-traffic-shifting的命名空间，并开启该空间内的istio-injection
   
   ```
   kubectl create namespace istio-io-tcp-traffic-shifting
   kubectl label namespace istio-io-tcp-traffic-shifting istio-injection=enabled
   ```

2. 部署sleep.yaml示例，作为请求发送源头
   
   ```
   kubectl apply -f samples/sleep/sleep.yaml -n istio-io-tcp-traffic-shifting 
   ```

3. 部署v1以及v2版本的tcp-echo微服务
   
   ```
   kubectl apply -f samples/tcp-echo/tcp-echo-services.yaml -n istio-io-tcp-traffic-shifting
   ```

4. 获取ingress IP以及ingress ports
   
   ```
   export TCP_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="tcp")].nodePort}')
   export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
   ```

   > 注：对于1.9.1版本的Istio而言，ingressgateway并未对tcp协议进行配置，此为一个issue，详见此[链接](https://github.com/istio/istio/issues/19995)  
   > 为使后续实验可以正常进行，需将tcp协议添加至ingressgateway中
   >
   > ```yaml
   >  - name: tcp
   >    nodePort: 31400
   >    port: 31400
   >    protocol: TCP
   >    targetPort: 31400
   > ```

## Apply weight-based TCP routing

1. 将所有的TCP流量配置为发送至tcp-echo微服务的v1版本
   
   ```
   kubectl apply -f samples/tcp-echo/tcp-echo-all-v1.yaml -n istio-io-tcp-traffic-shifting
   ```

2. 确认tcp-echo的服务已经启动，执行如下命令
   
   ```
   for i in {1..20}; do \
   kubectl exec "$(kubectl get pod -l app=sleep -n istio-io-tcp-traffic-shifting -o jsonpath={.items..metadata.name})" \
   -c sleep -n istio-io-tcp-traffic-shifting -- sh -c "(date; sleep 1) | nc $INGRESS_HOST $TCP_INGRESS_PORT"; \
   done
   ```

   输出内容如下

   ```
   one Wed Mar 31 08:53:06 UTC 2021
   one Wed Mar 31 08:53:07 UTC 2021
   one Wed Mar 31 08:53:08 UTC 2021
   one Wed Mar 31 08:53:10 UTC 2021
   one Wed Mar 31 08:53:11 UTC 2021
   one Wed Mar 31 08:53:13 UTC 2021
   one Wed Mar 31 08:53:14 UTC 2021
   one Wed Mar 31 08:53:15 UTC 2021
   one Wed Mar 31 08:53:17 UTC 2021
   one Wed Mar 31 08:53:18 UTC 2021
   one Wed Mar 31 08:53:19 UTC 2021
   one Wed Mar 31 08:53:21 UTC 2021
   one Wed Mar 31 08:53:22 UTC 2021
   one Wed Mar 31 08:53:24 UTC 2021
   one Wed Mar 31 08:53:25 UTC 2021
   one Wed Mar 31 08:53:26 UTC 2021
   one Wed Mar 31 08:53:28 UTC 2021
   one Wed Mar 31 08:53:29 UTC 2021
   one Wed Mar 31 08:53:30 UTC 2021
   one Wed Mar 31 08:53:32 UTC 2021
   ```

3. 使用如下命令，将20%流量从tcp-echo:v1转移至tcp-echo:v2
   
   ```
   kubectl apply -f samples/tcp-echo/tcp-echo-20-v2.yaml -n istio-io-tcp-traffic-shifting
   ```

4. 确认规则生效
   
   ```
   kubectl get virtualservices.networking.istio.io -n istio-io-tcp-traffic-shifting tcp-echo -o yaml
   ......
   spec:
     gateways:
     - tcp-echo-gateway
     hosts:
     - '*'
     tcp:
     - match:
       - port: 31400
       route:
       - destination:
           host: tcp-echo
           port:
             number: 9000
           subset: v1
         weight: 80
       - destination:
           host: tcp-echo
           port:
             number: 9000
           subset: v2
         weight: 20
   ```

5. 执行2中的命令
   
   ```
   for i in {1..20}; do \
   kubectl exec "$(kubectl get pod -l app=sleep -n istio-io-tcp-traffic-shifting -o jsonpath={.items..metadata.name})" \
   -c sleep -n istio-io-tcp-traffic-shifting -- sh -c "(date; sleep 1) | nc $INGRESS_HOST $TCP_INGRESS_PORT"; \
   done
   two Wed Mar 31 09:01:01 UTC 2021
   one Wed Mar 31 09:01:02 UTC 2021
   one Wed Mar 31 09:01:04 UTC 2021
   one Wed Mar 31 09:01:05 UTC 2021
   one Wed Mar 31 09:01:07 UTC 2021
   one Wed Mar 31 09:01:08 UTC 2021
   two Wed Mar 31 09:01:09 UTC 2021
   two Wed Mar 31 09:01:11 UTC 2021
   two Wed Mar 31 09:01:12 UTC 2021
   one Wed Mar 31 09:01:14 UTC 2021
   one Wed Mar 31 09:01:15 UTC 2021
   one Wed Mar 31 09:01:16 UTC 2021
   one Wed Mar 31 09:01:18 UTC 2021
   one Wed Mar 31 09:01:19 UTC 2021
   one Wed Mar 31 09:01:20 UTC 2021
   two Wed Mar 31 09:01:22 UTC 2021
   one Wed Mar 31 09:01:23 UTC 2021
   one Wed Mar 31 09:01:24 UTC 2021
   one Wed Mar 31 09:01:26 UTC 2021
   one Wed Mar 31 09:01:27 UTC 2021
   ```

   可以发现有20%的流量从tcp-echo:v1转移至tcp-echo:v2中

## Understanding what happened

In this task you partially migrated TCP traffic from an old to new version of the tcp-echo service using Istio’s weighted routing feature. Note that this is very different than doing version migration using the deployment features of container orchestration platforms, which use instance scaling to manage the traffic.

With Istio, you can allow the two versions of the tcp-echo service to scale up and down independently, without affecting the traffic distribution between them.
