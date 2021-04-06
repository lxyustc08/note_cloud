- [Mirroring](#mirroring)
  - [Before Testing](#before-testing)
  - [Creating a default routing policy](#creating-a-default-routing-policy)
  - [Mirroring traffic to v2](#mirroring-traffic-to-v2)

# Mirroring

本部分用于演示Istio的流量mirroring特性

流量镜像又被称为流量隐藏，通过该特性允许团队进行产品变更时以尽可能小的风险进行。流量镜像将实时流量副本发送至镜像服务。流量镜像发生在主服务关键请求路径带外，终端用户在使用过程中不会受到影响。

本次实验中，将首先将所有的流量配置到test service的v1版本，然后再应用一个将部分流量镜像到v2版本的服务

## Before Testing

本实验测试之前，需要进行如下准备：

+ 配置安装Istio
+ 部署两个版本的httpbin service，均开启两个服务的logging：
  
  v1 版本
  
  ```bash
  cat <<EOF | istioctl kube-inject -f - | kubectl create -f -
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: httpbin-v1
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: httpbin
        version: v1
    template:
      metadata:
        labels:
          app: httpbin
          version: v1
      spec:
        containers:
        - image: docker.io/kennethreitz/httpbin
          imagePullPolicy: IfNotPresent
          name: httpbin
          command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
          ports:
          - containerPort: 80
  EOF
  ```

  v2 版本

  ```bash
  cat <<EOF | istioctl kube-inject -f - | kubectl create -f -
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: httpbin-v2
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: httpbin
        version: v2
    template:
      metadata:
        labels:
          app: httpbin
          version: v2
      spec:
        containers:
        - image: docker.io/kennethreitz/httpbin
          imagePullPolicy: IfNotPresent
          name: httpbin
          command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
          ports:
          - containerPort: 80
  EOF
  ```

  httpbin kubernetes service:

  ```bash
  kubectl create -f - <<EOF
  apiVersion: v1
  kind: Service
  metadata:
    name: httpbin
    labels:
      app: httpbin
  spec:
    ports:
    - name: http
      port: 8000
      targetPort: 80
    selector:
      app: httpbin
  EOF
  ```

+ 启动sleep service，通过sleep service使用curl加载负载
  
  ```bash
  cat <<EOF | istioctl kube-inject -f - | kubectl create -f -
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: sleep
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: sleep
    template:
      metadata:
        labels:
          app: sleep
      spec:
        containers:
        - name: sleep
          image: tutum/curl
          command: ["/bin/sleep","infinity"]
          imagePullPolicy: IfNotPresent
  EOF
  ```

## Creating a default routing policy

默认情况下，kubernetes负载机制将流量均分到所有版本的httpbin service中，本处先创建一个Istio的流量规则，将所有流量引导至版本v1中。

1. 创建一个将所有流量引导至v1版本服务的规则
   
   ```
   kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: httpbin
   spec:
     hosts:
       - httpbin
     http:
     - route:
       - destination:
           host: httpbin
           subset: v1
         weight: 100
   ---
   apiVersion: networking.istio.io/v1alpha3
   kind: DestinationRule
   metadata:
     name: httpbin
   spec:
     host: httpbin
     subsets:
     - name: v1
       labels:
         version: v1
     - name: v2
       labels:
         version: v2
   EOF
   ```

2. 发送流量进行验证
   
   ```
   export SLEEP_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
   kubectl exec "${SLEEP_POD}" -c sleep -- curl -sS http://httpbin:8000/headers 

   {
     "headers": {
       "Accept": "*/*",
       "Content-Length": "0",
       "Host": "httpbin:8000",
       "User-Agent": "curl/7.35.0",
       "X-B3-Parentspanid": "8e9689a2173d2136",
       "X-B3-Sampled": "0",
       "X-B3-Spanid": "c136638a9473aed8",
       "X-B3-Traceid": "01828bb828cc952b8e9689a2173d2136",
       "X-Envoy-Attempt-Count": "1",
       "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/default;Hash=94c73c1d91217a8f9699e4348a82923e803a8a5d80797e75db8e3b77dabf4fae;Subject=\"\";URI=spiffe://cluster.local/ns/default/sa/default"
     }
   }
   ```

3. 检查httpbin的v1版本与v2版本的pods日志。
   
   v1版本日志
   
   ```
   export V1_POD=$(kubectl get pod -l app=httpbin,version=v1 -o jsonpath={.items..metadata.name})
   kubectl logs "$V1_POD" -c httpbin

   [2021-04-06 07:28:39 +0000] [1] [INFO] Starting gunicorn 19.9.0
   [2021-04-06 07:28:39 +0000] [1] [INFO] Listening at: http://0.0.0.0:80 (1)
   [2021-04-06 07:28:39 +0000] [1] [INFO] Using worker: sync
   [2021-04-06 07:28:39 +0000] [9] [INFO] Booting worker with pid: 9
   127.0.0.1 - - [06/Apr/2021:07:48:58 +0000] "GET /headers HTTP/1.1" 200 553 "-" "curl/7.35.0"
   127.0.0.1 - - [06/Apr/2021:07:49:05 +0000] "GET /headers HTTP/1.1" 200 553 "-" "curl/7.35.0"
   ```

   v2版本日志

   ```
   [2021-04-06 07:29:32 +0000] [1] [INFO] Starting gunicorn 19.9.0
   [2021-04-06 07:29:32 +0000] [1] [INFO] Listening at: http://0.0.0.0:80 (1)
   [2021-04-06 07:29:32 +0000] [1] [INFO] Using worker: sync
   [2021-04-06 07:29:32 +0000] [9] [INFO] Booting worker with pid: 9
   ```

   观察到，所有流量均分发到v1版本，v2版本中无流量进入

## Mirroring traffic to v2

1. 修改流量规则，将流量镜像到v2版本
   
   ```
   kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: httpbin
   spec:
     hosts:
       - httpbin
     http:
     - route:
       - destination:
           host: httpbin
           subset: v1
         weight: 100
       mirror:
         host: httpbin
         subset: v2
       mirrorPercent: 100
   EOF
   ```

   修改后的流量规则，将100%发送至v1的流量镜像至v2。当流量被镜像后，发送者镜像服务的请求的Host/Authority头将附加-shadow。比如cluster-1变为cluster-1-shadow。

2. 通过sleep发送流量
   
   ```
   kubectl exec "${SLEEP_POD}" -c sleep -- curl -sS http://httpbin:8000/headers
   ```

   查看两个版本的流量，v1版本

   ```
   kubectl logs "$V1_POD" -c httpbin
   127.0.0.1 - - [06/Apr/2021:08:19:59 +0000] "GET /headers HTTP/1.1" 200 553 "-" "curl/7.35.0"
   ```

   v2版本

   ```
   kubectl logs "$V2_POD" -c httpbin
   127.0.0.1 - - [06/Apr/2021:08:19:59 +0000] "GET /headers HTTP/1.1" 200 593 "-" "curl/7.35.0"
   ```

   观察可发现，流量已经镜像至v2版本