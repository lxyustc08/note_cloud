
- [Actions in accessing clusters](#actions-in-accessing-clusters)
  - [Directly accessing the REST API](#directly-accessing-the-rest-api)
    - [代理模式下使用Kubectl](#%e4%bb%a3%e7%90%86%e6%a8%a1%e5%bc%8f%e4%b8%8b%e4%bd%bf%e7%94%a8kubectl)
    - [不使用代理模式](#%e4%b8%8d%e4%bd%bf%e7%94%a8%e4%bb%a3%e7%90%86%e6%a8%a1%e5%bc%8f)
      - [使用kubectl describe secret获取token](#%e4%bd%bf%e7%94%a8kubectl-describe-secret%e8%8e%b7%e5%8f%96token)
      - [使用jsonpath获取token](#%e4%bd%bf%e7%94%a8jsonpath%e8%8e%b7%e5%8f%96token)
      - [相关说明](#%e7%9b%b8%e5%85%b3%e8%af%b4%e6%98%8e)

# Actions in accessing clusters

访问kubernetes集群的前提条件有两个：

+ 获取集群的位置
+ 拥有访问该集群的凭证

第一次使用kubernetes，建议使用命令行程序`kubectl`

## Directly accessing the REST API

`kubectl`命令负责处理*apiserver*的定位以及认证，在通过`curl`、`wget`或者`浏览器`等客户端使用RESTful风格的API时，通常有下列几种方式处理*apiserver*的<span style="border-bottom: 2px solid red; font-weight: bold">定位及认证</span>

+ 在代理模式下使用kubectl
  + kubernetes官方建议的方式
  + apiserver位置已被预先存储
  + 使用自签署证书验证apiserver身份，无中间人攻击（MITM）的可能性
  + 由apiserver验证身份
  + 后续迭代版本，将有智能化的客户端方向的负载均衡及失效机制
+ 直接在http请求中附带集群位置信息与集群访问凭证信息
  + 备用的机制
  + 在代理模式拒绝访问的情况下可正常工作
  + 需要在浏览器或客户端中导入根证书，以保护中间人攻击(MITM)

### 代理模式下使用Kubectl

使用如下命令启动`Kubectl`代理模式，代理模式下，`kubectl`充当反向代理服务器角色

```terminal
kubectl proxy --port=8080
```

`kubectl proxy`命令在<span style="border-bottom: 2px solid red; font-weight: bold">本地</span>*localhost*与*kubernetes API server*间创建一个代理服务器或应用层网关，<span style="border-bottom: 2px solid red; font-weight: bold">因此同样允许特定的静态流量通过指定的HTTP路径。除了指定的静态流量路径外，其他的输入数据通过一个指定的端口，然后由代理转发至远端的*kubernetes API server*上。</span>

> Creates a proxy server or application-level gateway between localhost and the Kubernetes API Server. It also allows serving static content over specified HTTP path. All incoming data enters through one port and gets forwarded to the remote kubernetes API Server port, except for the path matching the static content path.

在本地机器上使用如下命令

```terminal
curl http://localhost:8080/api/
```

输出如下结果

```terminal
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.10.197.97:6443"
    }
  ]
}
```

### 不使用代理模式

#### 使用kubectl describe secret获取token

1. 获取*apiserver*的位置  
   使用如下命令  
   ```terminal
   APISERVER=$(kubectl config view --minify | grep server | cut -f 2- -d ":" | tr -d " ")
   ```
2. 获取*secret*名称  
   使用如下命令
   ```terminal
   SECRET_NAME=$(kubectl get secrets | grep ^default | cut -f1 -d ' ')
   ```
3. 获取*token*  
   使用如下命令
   ```terminal
   TOKEN=$(kubectl describe secret $SECRET_NAME | grep -E '^token' | cut -f2 -d':' | tr -d " ")
   ```
4. 访问集群  
   在集群外且与集群联通的机器上10.10.197.133上，使用如下命令：
   ```terminal
   curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure
   ```
   > 此处的变量已经导出至10.10.197.133中
   输出如下：
   ```terminal
   {
     "kind": "APIVersions",
     "versions": [
       "v1"
     ],
     "serverAddressByClientCIDRs": [
       {
         "clientCIDR": "0.0.0.0/0",
         "serverAddress": "10.10.197.97:6443"
       }
     ]
   }
   ```

#### 使用jsonpath获取token

> jsonpath类似与Xpath，jsonpath的表达式通常用来路径检索或设置Json。表达式形式可以是`.`的形式或者`[]`形式。比如`$.store.book[0].title`和`$[‘store’][‘book’][0][‘title’]`，详细信息参考链接 https://blog.csdn.net/koflance/article/details/63262484   

1. 获取*apiserver*位置   
   ```terminal
   APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
   ```
2. 获取*secret*名称
   ```terminal
   SECRET_NAME=$(kubectl get serviceaccount default -o jsonpath='{.secrets[0].name}')
   ```
3. 获取*token*
   ```terminal
   TOKEN=$(kubectl get secret $SECRET_NAME -o jsonpath='{.data.token}' | base64 --decode)
   ```
   > 以json格式输出的经过了base64编码，需要解码base64 -decode
4. 在集群外的机器10.10.197.133上使用命令
   ```
   curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure
   ```
   > 此处的变量已经导出至10.10.197.133中
   ```terminal
   {
     "kind": "APIVersions",
     "versions": [
       "v1"
     ],
     "serverAddressByClientCIDRs": [
       {
         "clientCIDR": "0.0.0.0/0",
         "serverAddress": "10.10.197.97:6443"
       }
     ]
   }
   ```

#### 相关说明
在不使用代理模式时，使用了curl命令的--insecure参数，该参数不会检查证书的有效性。

> curl命令的描述 (TLS) By default, every SSL connection curl makes is verified to be secure. This option allows curl to proceed and operate even for server connections otherwise considered insecure.  
> The server connection is verified by making sure the server's certificate contains the right name and verifies successfully using the cert store.

去掉--insecure参数，输出结果如下
```terminal
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```
