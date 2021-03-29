- [Bookinfo Example Archtecture](#bookinfo-example-archtecture)
  - [Deploying application](#deploying-application)
    - [Before deploying application](#before-deploying-application)
    - [Deploying application](#deploying-application-1)
    - [Determin the ingress IP and port](#determin-the-ingress-ip-and-port)

# Bookinfo Example Archtecture

Bookinfo example是一个很好的演示用例，其基本结构如下所示：

![Alt Text](../Istio_picture/bookinfo_application.svg)

该演示用例由4个独立的微服务框架组成：

+ productpage.
  + calls the details and reviews microservices to populate the page
+ details.
  + contains book information
+ reviews.
  + contains book reviews, it also calls the rating microservice
+ ratings.
  + the ratings microservice contains book ranking information that accompanies a book review.

微服务reviews有3个版本：

+ Version v1 doesn't call the ratings service
+ Version v2 calls the ratings service, and displays each rating as 1 to 5 black starts.
+ Version v3 calls the ratings service, and displays each rating as 1 to 5 red starts.

## Deploying application

运行该samples无需对应用本身进行修改，用户仅需在一个Istio-enabled环境下配置运行该服务即可，此时Envoy sidecars将自动与每个服务进行交互。如下图所示：

![Alt Text](../Istio_picture/bookinfo_application_with_istio.svg)

### Before deploying application

使用demo以及default配置安装Istio时，开启了automatic sidecar injection配置，此时需要对namespace进行设置，打上istio-injection=enabled标签

```
kubectl label namespace default istio-injection=enabled
```

### Deploying application

使用如下命令部署bookinfo application

```
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

若关闭sidecar自动注入，使用如下命令

```
kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)
```

部署完后确认服务是否全部启动

```
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>
```

### Determin the ingress IP and port

1. 部署istio gateway
   
   ```
   kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
   ```

2. 确认gateway成功创建
   
   ```
   kubectl get gateway
   NAME               AGE
   bookinfo-gateway   4d4h
   ```

3. 配置node port，使服务可以从外部网络访问
   
   ```
   export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
   export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
   export TCP_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="tcp")].nodePort}')
   export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
   ```

4. 设置GATEWAY_URL
   
   ```
   export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
   ```

5. 确认是否可以外网访问
   
   ```
   curl -s "http://${GATEWAY_URL}/productpage" | grep -o "<title>.*</title>"
   <title>Simple Bookstore App</title>
   ```
