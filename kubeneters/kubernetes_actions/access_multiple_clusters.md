- [Configure Access to Multiple Clusters](#configure-access-to-multiple-clusters)
  - [Define cluster, users, and contexts](#define-cluster-users-and-contexts)

# Configure Access to Multiple Clusters

通过在配置文件中指定`clusters`, `users`, and `contexts`等属性，用户可以通过命令`kubectl config use-context`快速切换集群。

## Define cluster, users, and contexts

假设有两个集群

<table>
    <tr>
        <th>cluster name</th>
        <th>namespace name</th>
        <th>developer's name</th>
    </tr>
    <tr>
        <td>development</td>
        <td>frontend</td>
        <td>frontend developers</td>
    </tr>
    <tr>
        <td>development</td>
        <td>storage</td>
        <td>storage developers</td>
    </tr>
    <tr>
        <td>scratch</td>
        <td>default</td>
        <td>developers</td>
    </tr>
</table>


