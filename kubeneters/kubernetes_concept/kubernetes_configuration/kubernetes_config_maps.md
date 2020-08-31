- [ConfigMaps](#configmaps)

# ConfigMaps

一个`ConfigMap`是一个API对象，用以存储非机密的键值对信息，Pods可将ConfigMap作为环境变量(`environment variables`)，命令行变量(`command line variables`)以及存储在volume中的`configuration files`。

使用ConfigMap可将环境相关的配置（`environment specific configuration`）与`container images`解耦，这样可方便应用移植

**NOTE: `ConfigMap`存储的键值对信息是以明文方式存储，不提供任何加密机制，若存储的信息为加密信息，建议使用`Secrets`或其他第三方工具**


