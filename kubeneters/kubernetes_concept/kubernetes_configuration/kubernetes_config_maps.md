- [ConfigMaps](#configmaps)
  - [ConfigMap object](#configmap-object)
  - [ConfigMaps and Pods](#configmaps-and-pods)
  - [Using ConfigMaps](#using-configmaps)
    - [Using ConfigMaps as files from a Pod](#using-configmaps-as-files-from-a-pod)
      - [Mounted ConfigMaps are updated automatically](#mounted-configmaps-are-updated-automatically)
        - [FEATURE STATE: Kubernetes v1.19 [beta]](#feature-state-kubernetes-v119-beta)

# ConfigMaps

一个`ConfigMap`是一个API对象，用以存储非机密的键值对信息，Pods可将ConfigMap作为环境变量(`environment variables`)，命令行变量(`command line variables`)以及存储在volume中的`configuration files`。

使用ConfigMap可将环境相关的配置（`environment specific configuration`）与`container images`解耦，这样可方便应用移植

> **NOTE: `ConfigMap`存储的键值对信息是以明文方式存储，不提供任何加密机制，若存储的信息为加密信息，建议使用`Secrets`或其他第三方工具**

## ConfigMap object

***

一个`ConfigMap`对象与大多数的`Kubernetes Object`存在差异，他无`spec`域，不过其有`data`域，该`data`域存储items（key）以及对应的values

## ConfigMaps and Pods

***

在Pod的`spec`域中可指定引用的`ConfigMaps`，通过这种方式实现基于指定的引用的`ConfigMaps`存储的信息编辑Pods中运行的`containers`的目的。

> **NOTE: 通常情况下`ConfigMap`与`Pod`必须位于同一个的`namespace`中**

一个典型的`ConfigMap`对象，该对象中既有单行的属性值，又有多行的属性值

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # property-like keys; each key maps to a simple value
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"
  #
  # file-like keys
  game.properties: /
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: /
    color.good=purple
    color.bad=yellow
    allow.textmode=true
```

使用`ConfigMap`来配置`Pod`中的`contianer`有四种方式

+ 作为`container`的`entrypoint`使用的命令行参数
+ 作为`container`的环境变量
+ 作为一个只读的volume被`container`中的application使用
+ 在`Pod`中的`contianer`中编写代码，使用Kubernetes API读取`ConfigMap`

前三种使用方式中，`kubelet`在启动container时读取`ConfigMap`

第四种使用方式允许Pod中的container动态获取ConfigMap，从而保证**实时**使用最新版本的ConfigMap，**这种情况下，可通过Kubernetes API实现跨namespace使用。**

一个典型的使用game-demo `ConfigMap`对象的Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: game.example/demo-game
      env:
        # Define the environment variable
        - name: PLAYER_INITIAL_LIVES # Notice that the case is different here
                                     # from the key name in the ConfigMap.
          valueFrom:
            configMapKeyRef:
              name: game-demo           # The ConfigMap this value comes from.
              key: player_initial_lives # The key to fetch.
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
    # You set volumes at the Pod level, then mount them into containers inside that Pod
    - name: config
      configMap:
        # Provide the name of the ConfigMap you want to mount.
        name: game-demo
        # An array of keys from the ConfigMap to create as files
        items:
        - key: "game.properties"
          path: "game.properties"
        - key: "user-interface.properties"
          path: "user-interface.properties"
```

上述列子中，Pod的`spec`属性中定义了名称为`config`的volume，该volume被挂载至容器demo的`/config`路径下，并配置为只读属性。同时，volumes使用名称为game-demo的`ConfigMap`对象的两个key分别为`game.properties`与`user-interface.properties`，并将两个key值生成对应的`/config/user-interface.properties`与`/config/game.properties`文件，供container使用。如果在指定items时忽略key值，则默认将所有key对应的文件全部创建为文件。

## Using ConfigMaps

***

`ConfigMaps`可作为data volumes使用。此外，`ConfigMaps`可被系统其他部分使用，而无需直接导入给Pod，比如，`ConfigMaps`可存储系统其他部分所需使用的配置数据。

> Note:  
> 最常用的使用方式仍然是作为配置设置供container运行使用

### Using ConfigMaps as files from a Pod

在Pod的volume中使用ConfigMap，进行如下步骤：

1. 创建一个新的ConfigMap或使用现有的ConifgMap。多个Pods可引用同一个ConfigMap
2. 修改Pod定义，在`.spec.volumes[]`添加一个volume。volume名称随意命名，在`.spec.volumes[].configMap.name`域中指定引用的ConfigMap对象
3. 对于需要使用ConfigMaps的每个containers，添加`.spec.containers[].volumeMounts[]`，配置域`.spec.containers[].volumeMounts[].readOnly = true`并将域`.spec.containers[].volumeMounts[].mountPath`指向未使用的文件夹
4. 修改images或者command line以使用上述步骤中配置的文件

一个典型Pod使用ConfigMap放置在volume中

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    configMap:
      name: myconfigmap
```

#### Mounted ConfigMaps are updated automatically

当通过volume形式使用的`ConfigMap`升级后，即使是只读的keys同样会自动升级。kubelet会在每个周期性检查过程中探测`ConfigMap`是否刷新（kubelet的同步时延）。

由于缓存机制的存在，kubelet永远使用本地缓存来获取ConfigMap的值。缓存机制可通过配置KubeletConfiguration struct中的`ConfigMapAndSecretChangeDetectionStrategy`域来进行修改。对于ConfigMap而言，其可以通过watch(default)，ttl-based，或者直接重定向所有请求至API server来传播其值的变化状态。

**ConfigMap更新后，新的值传输到Pod的延迟为kubelet的同步时延+缓存的传播时延两部分组成，缓存的传播时延取决于选择的缓存类型**

##### FEATURE STATE: Kubernetes v1.19 [beta]

Kubernetes添加了immutable机制，通过该机制独立设置每个`Secrets`与`ConfigMaps`对象是否保持不变，避免出现误操作导致`Secrets`与`ConfigMaps`对象不受控改变，引起的服务故障。具体优势有两个：

+ 避免误操作引起的应用服务故障
+ 提高集群性能，通过Immutable，关闭kube apisever对特定`Secrets`与`ConfigMaps`对象的监测，从而提高kube apiserver的服务器性能

要使用上述特性，在feature gate中开启`ImmutableEphemeralVolumes`特性，并在`Secrets`与`ConfigMaps`对象中添加immutable域，如下所示

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  ...
data:
  ...
immutable: true
```

> Note:  
> 上述修改步骤不可逆，一旦指定为immutable，后续无法再改变回普通对象，改变相关值的唯一的方式是删除相关的对象，并重新创建。此时使用到对象的Pods的挂载点需同步删除，并重新创建Pods
