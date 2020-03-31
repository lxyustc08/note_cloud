- [Organizing Cluster Access Using Kubeconfig Files](#organizing-cluster-access-using-kubeconfig-files)
  - [Supporting multiple clusters, users, and authentication mechanisms](#supporting-multiple-clusters-users-and-authentication-mechanisms)
  - [Context](#context)
  - [The KUBECONFIG environment variable](#the-kubeconfig-environment-variable)
  - [Merging kubeconfig files](#merging-kubeconfig-files)
  - [File references](#file-references)

# Organizing Cluster Access Using Kubeconfig Files

**kubeconfig files**被用来组织关于`clusters`, `users`, `namespaces`以及`authentication mechanism`相关信息。`kubectl` 命令行工具同样使用kubeconfig files寻找其选择的集群以及与该集群的API server进行通信的信息。

> 注意，此处的kubeconfig file指的是用以配置集群访问信息的文件，并不意味着该文件的名字就是kubeconfig

默认情况下，`kubectl`寻找路径`$HOME/.kube`下名称为`config`的文件，当然可以通过配置环境变量$KUBECONFIG或者kubectl的参数--kubeconfig指定其他的kubeconfig files

## Supporting multiple clusters, users, and authentication mechanisms

假定你有多个集群，每个集群的用户和组件认证通过多种方式进行，比如：

+ A running kubelet might authenticate using certificates
+ A user might authenticate using tokens
+ Administrtors might have sets of certificates that they provide to individual users

通过使用kubeconfig files，可以组织`clusters`, `users`以及`namespaces`。而且可以通过定义`contexts`快速切换`clusters`以及`namespaces`

## Context

kubeconfig file中的`context`元素用于将访问参数组织成一个便于使用的名字。每个context有三个参数：`cluster`, `namespace`以及`user`。默认情况下，kubectl命令行从`current context`中获取参数与集群通信。

## The KUBECONFIG environment variable

`KUBECONFIG`环境变量存储kubeconfig files的列表，Linux和MAC下，列表使用`:`分割，Windows下列表使用`;`分割。

环境变量`KUBECONFIG`，不是必需的，若其不存在，kubectl使用默认的kubeconfig file，`$HOME/.kube/config`。

如果环境变量`KUBECONFIG`存在，kubectl使用环境变量`KUBECONFIG`指向的列表合并后的有效值作为有效的配置文件。

## Merging kubeconfig files

命令
```terminal
kubectl config view
```
输出的结果有可能不是来自于单个kubeconfig文件，有可能来自于多个kubeconfig文件拼接合并而来，关于kubeconfig file，其合并遵循如下规则：

+ 如果`kubectl`指定了`--kubeconfig`参数，则使用该参数指定的唯一kubeconfig文件
+ 如果环境变量`KUBECONFIG`被指定，则将环境变量`KUBECONFIG`指定的kubeconfig files进行合并，合并过程遵循如下规则：
  + ignore empty filenames
  + Produce errors for files with content that cannot be deserialized (反序列化)
  + The first file to set a particular value or map key wins（先到先得）
  + Never change the value or map key（已存在的值不进行修改）
+ 无上述两种情况，使用默认的路径`$HOME/.kube/config`下默认的kubeconfig file，无合并行为

对于kubectl命令行工具,关于context的选择基于下列规则，优先选择符合条件的:

1. Use the `--context` command-line flag if it exits.
2. Use the `current-context` from the merged kubeconfig files.

对于kubectl命令行工具,空context在下述情况下是被允许的：
**Determine the cluster and user**. At this point, there might or might not be a context. Determine the cluster and user based on the first hit in this chain, **which is run twice: once for user and once for cluster**:

1. Use a command-line flag if it exits: `--user` or `--cluster`
2. If the context is non-empty, take the user or cluster from the context

对于kubectl命令行工具,空user及空cluster在下述情况下是被允许的：

1. **Determine the actual cluster information to use**. At this point, there might or might not be cluster information. Build each piece of the cluster information based on this chain; the first hit wins:
   1. Use command line flags if they exist: `--server`, `--certificate-authority`, `--insecure-skip-tls-verify`.
   2. If any cluster information attributes exist from the merged kubeconfig files, use them.
2. **Determine the actual user information to use**. Build user information using the same rules as cluster information, **except allow only one authentication technique per user**:
   1. Use command line flags if they exist: `--client-certificate`, `--client-key`, `--username`, `--password`, `--token`.
   2. Use the user fields from the merged kubeconfig files.
   3. If there are two conflicting techniques, fail.
3. For any information still missing, use default values and potentially prompt for authentication information.

## File references

kubeconfig file中引用的文件路径相对于kubeconfig file位置路径而言，在`$HOME/.kube/config`文件中，相对路径相对存储，绝对路径绝对存储
