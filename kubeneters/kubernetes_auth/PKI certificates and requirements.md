- [PKI certificates and requirements](#pki-certificates-and-requirements)
	- [前置知识 Server certificate 和 Client certificate](#%e5%89%8d%e7%bd%ae%e7%9f%a5%e8%af%86-server-certificate-%e5%92%8c-client-certificate)
	- [Kubernetes证书使用方式](#kubernetes%e8%af%81%e4%b9%a6%e4%bd%bf%e7%94%a8%e6%96%b9%e5%bc%8f)
	- [Kubernetes证书存储位置](#kubernetes%e8%af%81%e4%b9%a6%e5%ad%98%e5%82%a8%e4%bd%8d%e7%bd%ae)
	- [手动配置证书](#%e6%89%8b%e5%8a%a8%e9%85%8d%e7%bd%ae%e8%af%81%e4%b9%a6)
		- [Single root CA](#single-root-ca)
		- [All certificates](#all-certificates)
			- [Certificate paths](#certificate-paths)
	- [Configure certificates for user accounts](#configure-certificates-for-user-accounts)

# PKI certificates and requirements

Kubernetes使用PKI证书进行TLS身份验证。使用工具`kubeadm`安装Kubernetes集群时，Kubernetes集群所需的证书会由工具`kubeadm`自动生成，当然用户可根据需求手动生成相关证书。

## 前置知识 Server certificate 和 Client certificate

+ Server certificate
  + 标识服务器身份，如安装在网站上的SSL证书。</span>。
+ Client cerfificate
  + 标识自己身份



## Kubernetes证书使用方式

Kubernetes使用证书对下列操作进行验证：

+ 对于`kubelet`而言，需要*client certificate*以向`API server`标识自己身份
  + 使用`kubeadm`部署的集群在每个node上的文件夹/etc/kubernetes/pki中存储该证书
+ 对于`API server endpoint`而言，需要*server cerfificate*标识自己的身份
+ 对于集群管理员而言，需要*client certificate*向*API server*标识自己身份
+ 对于`API server`向`kubelets`传输消息这一过程，需要使用*API server*的客户端证书，标识*API server*身份
+ 对于`API server`向`etcd`传输消息这一过程，需要使用*API server*的客户端证书，标识*API server*身份
+ 对于`controller manager`向`API server`传输消息这一过程，需要*client certificate*或*kubeconfig*标识自己身份
+ 对于`scheduler`向`API server`传输消息这一过程，需要*client certificate*或*kubeconfig*标识自身身份
+ 对于`front-proxy`而言，同时需要*client certificate*与*server certificate*

## Kubernetes证书存储位置

使用`kubeadm`部署Kubernetes集群时，证书默认存储位置为`/etc/kubernetes/pki`

## 手动配置证书

如果用户不想使用`kubeadm`生成的证书，也可以手动配置，手动配置证书可通过下列任一方法进行。

### Single root CA

用户可以创建一个单独的root CA，该CA由管理员控制。该root CA可以创建多个中间CA(`intermediate CAs`)，并将后续创建工作交给Kubernetes本身。

需要的CA

<table>
  <tr>
    <th>path</th>
    <th>Default CN (Common Name)</th>
    <th>description</th>
  </tr>
  <tr>
    <td>ca.crt, ca.key</td>
    <td>kubernetes-ca</td>
    <td>Kubernetes general CA</td>
  </tr>
  <tr>
    <td>etcd/ca.crt, etcd/ca.key</td>
    <td>etcd-ca</td>
    <td>For all etcd-related funcations</td>
  </tr>
  <tr>
    <td>front-proxy-ca.crt, front-proxy-ca.key</td>
    <td>kubernetes-front-proxy-ca</td>
    <td>For the front-end proxy</td>
  </tr>
</table>

在所有CA之上，需要密钥对管理service account，`sa.key`和`sa.pub`

### All certificates

同样，可以手动生成所有的证书

需要的证书

<table>
	<tr>
		<th>Default CN</th>
		<th>Parent CA</th>
		<th>O (in subject) Organization name</th>
		<th>kind</th>
		<th>hosts(SAN)</th>
	</tr>
	<tr>
		<td>kube-etcd</td>
		<td>etcd-ca</td>
		<td>  </td>
		<td>server, client</td>
		<td>localhost, 127.0.0.1</td>
	</tr>
	<tr>
		<td>kube-etcd-peer</td>
		<td>etcd-ca</td>
		<td>  </td>
		<td>server, client</td>
		<td>&lthostname&gt, &ltHOST_IP&gt, localhost, 127.0.0.1</td>
	</tr>
	<tr>
		<td>kube-etcd-healthcheck-client</td>
		<td>etcd-ca</td>
		<td>  </td>
		<td>client</td>
		<td>  </td>
		<td>  </td>
	</tr>
	<tr>
		<td>kube-apiserver-etcd-client</td>
		<td>etcd-ca</td>
		<td>system:masters</td>
		<td>client</td>
		<td></td>
	</tr>
	<tr>
		<td>kube-apiserver</td>
		<td>kubernetes-ca</td>
		<td></td>
		<td>server</td>
		<td>&lthostname&gt, &ltHOST_IP&gt, &ltadvertise_IP&gt, [1]</td>
	</tr>
	<tr>
		<td>kube-apiserver-kubelet-client</td>
		<td>kubernetes-ca</td>
		<td>system:masters</td>
		<td>client</td>
		<td></td>
	</tr>
	<tr>
		<td>front-proxy-client</td>
		<td>kubernetes-front-proxy-ca</td>
		<td></td>
		<td>client</td>
		<td></td>
	</tr>
</table>

> [1]: any other IP or DNS name you contact your cluster on (as used by kubeadm the load balancer stable IP and/or DNS name, kubernetes, kubernetes.default, kubernetes.default.svc, kubernetes.default.svc.cluster, kubernetes.default.svc.cluster.local)

> 对于kubeadm用户而言需要注意以下几点：  
> + The scenario where you are copying to your cluster CA certificates without private keys is referred as external CA in the kubeadm documentation.
> + If you are comparing the above list with a kubeadm generated PKI, please be aware that kube-etcd, kube-etcd-peer and kube-etcd-healthcheck-client certificates are not generated in case of external etcd.

#### Certificate paths

证书建议的存储路径如下：

<table>
	<tr>
		<th>Default CN</th>
		<th>recommended key path</th>
		<th>recommended cert path</th>
		<th>command</th>
		<th>key argument</th>
		<th>cert argument</th>
	</tr>
	<tr>
		<td>etcd-ca</td>
		<td>etcd/ca.key</td>
		<td>etcd/ca.crt</td>
		<td>kube-apiserver</td>
		<td></td>
		<td>–etcd-cafile</td>
	</tr>
	<tr>
		<td>kube-apiserver-etcd-client</td>
		<td>apiserver-etcd-client.key</td>
		<td>apiserver-etcd-client.crt</td>
		<td>kube-apiserver</td>
		<td>–etcd-keyfile</td>
		<td>–etcd-certfile</td>
	</tr>
	<tr>
		<td>kubernetes-ca</td>
		<td>ca.key</td>
		<td>ca.crt</td>
		<td>kube-apiserver</td>
		<td></td>
		<td>–client-ca-file</td>
	</tr>
	<tr>
		<td>kubernetes-ca</td>
		<td>ca.key</td>
		<td>ca.crt</td>
		<td>kube-controller-manager</td>
		<td>–cluster-signing-key-file</td>
		<td>–client-ca-file, –root-ca-file, –cluster-signing-cert-file</td>
	</tr>
	<tr>
		<td>kube-apiserver</td>
		<td>apiserver.key</td>
		<td>apiserver.crt</td>
		<td>kube-apiserver</td>
		<td>–tls-private-key-file</td>
		<td>–tls-cert-file</td>
	</tr>
	<tr>
		<td>kube-apiserver-kubelet-client</td>
		<td>apiserver-kubelet-client.key</td>
		<td>apiserver-kubelet-client.crt</td>
		<td>kube-apiserver</td>
		<td>–kubelet-client-key</td>
		<td>–kubelet-client-certificate</td>
	</tr>
	<tr>
		<td>front-proxy-ca</td>
		<td>front-proxy-ca.key</td>
		<td>front-proxy-ca.crt</td>
		<td>kube-apiserver</td>
		<td></td>
		<td>–requestheader-client-ca-file</td>
	</tr>
	<tr>
		<td>front-proxy-ca</td>
		<td>front-proxy-ca.key</td>
		<td>front-proxy-ca.crt</td>
		<td>kube-controller-manager</td>
		<td></td>
		<td>–requestheader-client-ca-file</td>
	</tr>
	<tr>
		<td>front-proxy-client</td>
		<td>front-proxy-client.key</td>
		<td>front-proxy-client.crt</td>
		<td>kube-apiserver</td>
		<td>–proxy-client-key-file</td>
		<td>–proxy-client-cert-file</td>
	</tr>
	<tr>
		<td>etcd-ca</td>
		<td>etcd/ca.key</td>
		<td>etcd/ca.crt</td>
		<td>etcd</td>
		<td></td>
		<td>–trusted-ca-file, –peer-trusted-ca-file</td>
	</tr>
	<tr>
		<td>kube-etcd</td>
		<td>etcd/server.key</td>
		<td>etcd/server.crt</td>
		<td>etcd</td>
		<td>–key-file</td>
		<td>–cert-file</td>
	</tr>
	<tr>
		<td>kube-etcd-peer</td>
		<td>etcd/peer.key</td>
		<td>etcd/peer.crt</td>
		<td>etcd</td>
		<td>–peer-key-file</td>
		<td>–peer-cert-file</td>
	</tr>
	<tr>
		<td>etcd-ca</td>
		<td></td>
		<td>etcd/ca.crt</td>
		<td>etcdctl</td>
		<td></td>
		<td>–cacert</td>
	</tr>
	<tr>
		<td>kube-etcd-healthcheck-client</td>
		<td>etcd/healthcheck-client.key</td>
		<td>etcd/healthcheck-client.crt</td>
		<td>etcdctl</td>
		<td>–key</td>
		<td>–cert</td>
	</tr>
</table>

对于`service account`而言，密钥对同样需要

<table>
	<tr>
		<th>private key path</th>
		<th>public key path</th>
		<th>command</th>
		<th>argument</th>
	</tr>
	<tr>
		<td>sa.key</td>
		<td></td>
		<td>kube-controller-manager</td>
		<td>service-account-private</td>
	</tr>
	<tr>
		<td></td>
		<td>sa.pub</td>
		<td>kube-apiserver</td>
		<td>service-account-key</td>
	</tr>
</table>

## Configure certificates for user accounts

部分`administrator account`和`service account`不可缺少，需要手动配置，当然除`kubeadm`之外，该工具自动创建相关证书

<table>
	<tr>
		<th>filename</th>
		<th>credential name</th>
		<th>Default CN</th>
		<th>O (in Subject)</th>
	</tr>
	<tr>
		<td>admin.conf</td>
		<td>default-admin</td>
		<td>kubernetes-admin</td>
		<td>system:masters</td>
	</tr>
	<tr>
		<td>kubelet.conf</td>
		<td>default-auth</td>
		<td>system:node:&ltnodeName&gt</td>
		<td>system:nodes</td>
	</tr>
	<tr>
		<td>controller-manager.conf</td>
		<td>default-controller-manager</td>
		<td>system:kube-controller-manager</td>
		<td></td>
	</tr>
	<tr>
		<td>scheduler.conf</td>
		<td>default-scheduler</td>
		<td>system:kube-scheduler</td>
		<td></td>
	</tr>
</table>

手动配置证书步骤：

1. 根据配置文件规定的CN和O字段，生成x509 cert/key对；
2. 针对每个配置文件运行`kubectl`命令
   ```terminal
   KUBECONFIG=<filename> kubectl config set-cluster default-cluster --server=https://<host ip>:6443 --certificate-authority <path-to-kubernetes-ca> --embed-certs
   KUBECONFIG=<filename> kubectl config set-credentials <credential-name> --client-key <path-to-key>.pem --client-certificate <path-to-cert>.pem --embed-certs
   KUBECONFIG=<filename> kubectl config set-context default-system --cluster default-cluster --user <credential-name>
   KUBECONFIG=<filename> kubectl config use-context default-system
   ```

上述文件在各命令中的使用如下

<table>
	<tr>
		<th>filename</th>
		<th>command</th>
		<th>comment</th>
	</tr>
	<tr>
		<td>admin.conf</td>
		<td>kubectl</td>
		<td>Configures administrator user for the cluster</td>
	</tr>
	<tr>
		<td>kubelet.conf</td>
		<td>kubelet</td>
		<td>One required for each node in the cluster.</td>
	</tr>
	<tr>
		<td>controller-manager.conf</td>
		<td>kube-controller-manager</td>
		<td>Must be added to manifest in manifests/kube-controller-manager.yaml</td>
	</tr>
	<tr>
		<td>scheduler.conf</td>
		<td>kube-scheduler</td>
		<td>Must be added to manifest in manifests/kube-scheduler.yaml</td>
	</tr>	
</table>
