- [Container Storage Interface](#container-storage-interface)
  - [Objective](#objective)
  - [Terminology](#terminology)
    - [Goals in MVP](#goals-in-mvp)
  - [SPEC OVERVIEW](#spec-overview)
  - [CSI Architect](#csi-architect)
    - [Centralized Controller Plugin Deployment](#centralized-controller-plugin-deployment)
    - [Headless Plugin Deployment](#headless-plugin-deployment)
      - [Split-components](#split-components)
      - [Unified-components](#unified-components)
      - [No Controller Plugins](#no-controller-plugins)
  - [Volume Lifecycle](#volume-lifecycle)
  - [Container Storage Interface](#container-storage-interface-1)
    - [RPC Interface](#rpc-interface)

# Container Storage Interface

## Objective

1. CSI was developed as a standard for exposing arbitrary block and file storage storage systems to containerized workloads on Container Orchestration Systems (COs) like Kubernetes.
   + **With the adoption of the Container Storage Interface, the Kubernetes volume layer becomes truly extensible.**
2. Using CSI, third-party storage providers can write and deploy plugins exposing new storage systems in Kubernetes without ever having to touch the core Kubernetes code
   + **This gives Kubernetes users more options for storage and makes the system more secure and reliable.**

## Terminology

|Term | Definition|
|:---:|:---|
|Volume|A unit of storage that will be made available inside of a CO-managed container, via the CSI.|
|Block Volume|A volume that will appear as a block device inside the container.|
|Mounted Volume|A volume that will be mounted using the specified file system and appear as a directory inside the container.|
|CO|Container Orchestration system, communicates with Plugins using CSI service RPCs.|
|SP|Storage Provider, the vendor of a CSI plugin implementation.|
|RPC|Remote Procedure Call.|
|Node|A host where the user workload will be running, uniquely identifiable from the perspective of a Plugin by a node ID.|
|Plugin|Aka “plugin implementation”, a gRPC endpoint that implements the CSI Services.|
|Plugin Supervisor|Process that governs the lifecycle of a Plugin, MAY be the CO.|
|Workload|The atomic unit of "work" scheduled by a CO. This MAY be a container or a collection of containers.|

### Goals in MVP

**CSI将做什么：**

+ Enable SP authors to write one CSI compliant Plugin that “just works” across all COs that implement CSI.
+ Define API (RPCs) that enable:
  + Dynamic provisioning and deprovisioning of a volume.
  + Attaching or detaching a volume from a node.
  + Mounting/unmounting a volume from a node.
  + Consumption of both block and mountable volumes.
  + Local storage providers (e.g., device mapper, lvm).
  + Creating and deleting a snapshot (source of the snapshot is a volume).
  + Provisioning a new volume from a snapshot **(reverting snapshot, where data in the original volume is erased and replaced with data in the snapshot, is out of scope)**.
+ Define plugin protocol RECOMMENDATIONS
  + Describe a process by which a Supervisor configures a Plugin.
  + Container deployment considerations (CAP_SYS_ADMIN, mount namespace, etc.).

**CSI将不做什么：**

+ Specific mechanisms by which a Plugin Supervisor manages the lifecycle of a Plugin, including:
  + How to maintain state (e.g. what is attached, mounted, etc.).
  + How to deploy, install, upgrade, uninstall, monitor, or respawn (in case of unexpected termination) Plugins.
+ A first class message structure/field to represent "grades of storage" (aka "storage class").
+ Protocol-level authentication and authorization.
+ Packaging of a Plugin.
+ POSIX compliance: CSI provides no guarantee that volumes provided are POSIX compliant filesystems. Compliance is determined by the Plugin implementation (and any backend storage system(s) upon which it depends). CSI SHALL NOT obstruct a Plugin Supervisor or CO from interacting with Plugin-managed volumes in a POSIX-compliant manner.

## SPEC OVERVIEW

This specification defines an interface along with the minimum operational and packaging recommendations for a storage provider (SP) to **implement a CSI compatible plugin**. **The interface declares the RPCs that a plugin MUST expose: this is the primary focus of the CSI specification**. Any operational and packaging **recommendations** offer additional guidance to **promote cross-CO compatibility**.

## CSI Architect


The primary **focus of this specification is on the protocol between a CO and a Plugin.** It **SHOULD** be possible to **ship cross-CO compatible Plugins for a variety of deployment architectures**. A **CO SHOULD** be equipped to **handle both centralized and headless plugins, as well as split-component and unified plugins.** Several of these possibilities are illustrated in the following figures.

### Centralized Controller Plugin Deployment

---

```
                             CO "Master" Host
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |     CO     |   gRPC    | Controller |  |
|  |            +----------->   Plugin   |  |
|  +------------+           +------------+  |
|                                           |
+-------------------------------------------+

                            CO "Node" Host(s)
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |     CO     |   gRPC    |    Node    |  |
|  |            +----------->   Plugin   |  |
|  +------------+           +------------+  |
|                                           |
+-------------------------------------------+

Figure 1: The Plugin runs on all nodes in the cluster: a centralized
Controller Plugin is available on the CO master host and the Node
Plugin is available on all of the CO Nodes.
```

### Headless Plugin Deployment

---

#### Split-components

```
                            CO "Node" Host(s)
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |     CO     |   gRPC    | Controller |  |
|  |            +--+-------->   Plugin   |  |
|  +------------+  |        +------------+  |
|                  |                        |
|                  |                        |
|                  |        +------------+  |
|                  |        |    Node    |  |
|                  +-------->   Plugin   |  |
|                           +------------+  |
|                                           |
+-------------------------------------------+

Figure 2: Headless Plugin deployment, only the CO Node hosts run
Plugins. Separate, split-component Plugins supply the Controller
Service and the Node Service respectively.
```

#### Unified-components

```
                            CO "Node" Host(s)
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |     CO     |   gRPC    | Controller |  |
|  |            +----------->    Node    |  |
|  +------------+           |   Plugin   |  |
|                           +------------+  |
|                                           |
+-------------------------------------------+

Figure 3: Headless Plugin deployment, only the CO Node hosts run
Plugins. A unified Plugin component supplies both the Controller
Service and Node Service.
```

#### No Controller Plugins

```
                            CO "Node" Host(s)
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |     CO     |   gRPC    |    Node    |  |
|  |            +----------->   Plugin   |  |
|  +------------+           +------------+  |
|                                           |
+-------------------------------------------+

Figure 4: Headless Plugin deployment, only the CO Node hosts run
Plugins. A Node-only Plugin component supplies only the Node Service.
Its GetPluginCapabilities RPC does not report the CONTROLLER_SERVICE
capability.
```

在无controller service plugins部署模式下，`GetPluginCapabilities` RPC调用**不会报告**CONTROLLER_SERVICE
capability

## Volume Lifecycle

```
   CreateVolume +------------+ DeleteVolume
 +------------->|  CREATED   +--------------+
 |              +---+----^---+              |
 |       Controller |    | Controller       v
+++         Publish |    | Unpublish       +++
|X|          Volume |    | Volume          | |
+-+             +---v----+---+             +-+
                | NODE_READY |
                +---+----^---+
               Node |    | Node
            Publish |    | Unpublish
             Volume |    | Volume
                +---v----+---+
                | PUBLISHED  |
                +------------+

Figure 5: The lifecycle of a dynamically provisioned volume, from
creation to destruction.
```

```
   CreateVolume +------------+ DeleteVolume
 +------------->|  CREATED   +--------------+
 |              +---+----^---+              |
 |       Controller |    | Controller       v
+++         Publish |    | Unpublish       +++
|X|          Volume |    | Volume          | |
+-+             +---v----+---+             +-+
                | NODE_READY |
                +---+----^---+
               Node |    | Node
              Stage |    | Unstage
             Volume |    | Volume
                +---v----+---+
                |  VOL_READY |
                +---+----^---+
               Node |    | Node
            Publish |    | Unpublish
             Volume |    | Volume
                +---v----+---+
                | PUBLISHED  |
                +------------+

Figure 6: The lifecycle of a dynamically provisioned volume, from
creation to destruction, when the Node Plugin advertises the
STAGE_UNSTAGE_VOLUME capability.
```

```
    Controller                  Controller
       Publish                  Unpublish
        Volume  +------------+  Volume
 +------------->+ NODE_READY +--------------+
 |              +---+----^---+              |
 |             Node |    | Node             v
+++         Publish |    | Unpublish       +++
|X| <-+      Volume |    | Volume          | |
+++   |         +---v----+---+             +-+
 |    |         | PUBLISHED  |
 |    |         +------------+
 +----+
   Validate
   Volume
   Capabilities

Figure 7: The lifecycle of a pre-provisioned volume that requires
controller to publish to a node (`ControllerPublishVolume`) prior to
publishing on the node (`NodePublishVolume`).
```

```
       +-+  +-+
       |X|  | |
       +++  +^+
        |    |
   Node |    | Node
Publish |    | Unpublish
 Volume |    | Volume
    +---v----+---+
    | PUBLISHED  |
    +------------+

Figure 8: Plugins MAY forego other lifecycle steps by contraindicating
them via the capabilities API. Interactions with the volumes of such
plugins is reduced to `NodePublishVolume` and `NodeUnpublishVolume`
calls.
```

The above diagrams illustrate a general expectation with respect to how a CO MAY manage the lifecycle of a volume via the API presented in this specification. **Plugins SHOULD expose all RPCs for an interface**: **Controller plugins SHOULD implement all RPCs for the Controller service. Unsupported RPCs SHOULD return an appropriate error code that indicates such (e.g. CALL_NOT_IMPLEMENTED)**. The full list of plugin capabilities is documented in the ControllerGetCapabilities and NodeGetCapabilities RPCs.

> 插件应该公开所有的RPC，控制插件应该实现所有的用以支持controller service的RPCs，对于插件而言，对于其不支持的RPCs，插件应该在接到该RPC时，返回一个恰当的错误代码

## Container Storage Interface

**Describes the interface between COs and Plugins.**

### RPC Interface

A CO interacts with an Plugin through RPCs. Each SP MUST provide:

+ **Node Plugin:** A gRPC endpoint serving CSI RPCs that MUST be run on the Node whereupon an SP-provisioned volume will be published.
+ **Controller Plugin:** A gRPC endpoint serving CSI RPCs that MAY be run anywhere.
+ In some circumstances a single gRPC endpoint MAY serve all CSI RPCs
