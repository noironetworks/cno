# Cisco Network Operator for Cisco Fabrics

# Table of Contents

- [1. Overview](#1-overview)
- [2. Features](#2-features)
- [3. Cisco Network Operator for Telco 5G cloud-native fabric orchestration](#3-cisco-network-operator-for-telco-5g-cloud-native-fabric-orchestration)
  - [3.1 Cisco ACI and Openshift node physical connectivity for CNO orchestration](#31-cisco-aci-and-openshift-node-physical-connectivity-for-cno-orchestration)
  - [3.2 Cisco ACI and Openshift cluster workloads logical network design](#32-cisco-aci-and-openshift-cluster-workloads-logical-network-design)
- [4 Network Operator software architecture](#4-network-operator-software-architecture)
- [5 Network Operator installation](#5-network-operator-installation)
  - [5.1 Network Operator installation pre-requisites](#51-network-operator-installation-pre-requisites)
  - [5.2. Network Operator installation](#52-network-operator-installation)
- [6. Network Operator - quick start guide](#6-network-operator---quick-start-guide)
  - [6.1 Network Operator - orchestrate fabric configuration for SR-IOV interfaces connected to POD](#61-network-operator---orchestrate-fabric-configuration-for-sr-iov-interfaces-connected-to-pod)
  - [6.2 Network Operator - orchestrate fabric configuration for MACVLAN interfaces connected to POD](#62-network-operator---orchestrate-fabric-configuration-for-macvlan-interfaces-connected-to-pod)
  - [6.3 Provisioning multiple VLANs within the same NetworkAttachmentDefinition](#63-provisioning-multiple-vlans-within-the-same-networkattachmentdefiniton)
  - [6.4 Network operator - orchestrate fabric configuration for IPVLAN interfaces connected to Pod](#64-orchestrate-fabric-configuration-for-IPVLAN-interfaces-connected-to-Pod)
  - [6.5 Network operator - orchestrate fabric configuration for Bridge interfaces connected to Pod](#65-orchestrate-fabric-configuration-for-bridgelinux-bridge-interfaces-connected-to-pod)
  - [6.6 Network operator - orchestrate fabric configuration for ovs interfaces connected to Pod](#66-orchestrate-fabric-configuration-for-ovs-bridge-interfaces-connected-to-pod)  
- [7. Webhook for adding CNO to NAD CNI chain](#7-webhook-for-adding-cno-into-the-nad-cni-chain)
  - [7.1 Provisioning CNO auto insertion webhook](#71-provisioning-webhook-for-cno-insertion-into-nad-cni-chain)
- [8. Custom features for Ericsson Packet Core support](#8-custom-features-for-ericsson-packet-core-support)
  - [8.1 Custom requirements](#81-custom-requirements)
  - [8.2 Network Operator implementation](#82-network-operator-implementation)
    - [8.2.1 NadVlanMap Custom Resource](#821-nadvlanmap-custom-resource)
    - [8.2.2 FabricVlanPool custom resource](#822-fabricvlanpool-custom-reosource)
    - [8.2.3 External router port attachment](#823-external-router-port-attachment)
    - [8.2.4 Custom EPG and BD Configuration](#824-Custom-EPG-and-BD-Configuration)
    - [8.2.5 VLAN file ingest](#825-vlan-file-ingest)
- [9. Primary CNI chaining (tech-preview)](#9-primary-cni-chaining-tech-preview)
- [10. Network Operator - orchestrate fabric for router pods using additional interfaces](#10-network-operator---orchestrate-fabric-for-router-pods-using-additional-interfaces)
  - [10.1 Using pre-existing ACI L3Out](#101-using-pre-existing-aci-l3out)
  - [10.2 Using generated ACI L3Out](#102-using-generated-aci-l3out)
  - [10.3 Kubernetes node to ACI leaf peer mapping](#103-kubernetes-node-to-aci-leaf-peer-mapping)
  - [10.4 Webhook based environment variable insertion for router pods ](#104-webhook-based-environment-variable-insertion-for-router-pods)
  - [10.5 Caveats](#105-Caveats)

## 1. Overview

Network Operator enables 5G platform operators to specify configuration intent for Container Network Function (CNF) interfaces natively using Kubernetes resources and annotations. 
Because the Network Operator is written as a fully compliant Kubernetes Operator, it continuously reconciles the state of the Cisco fabric, the Kubernetes scheduler, and each CNF interface configuration to ensure required network resources are provided and configured appropriately at each Kubernetes node so that CNFs can consume them.

Now, a 5G platform operator can dynamically reconfigure and scale CNFs and the Cisco Network Operator will ensure that the Network Fabric is provisioned and re-provisioned as necessary.

## 2. Features
This release includes the following features:

* An Extensible Framework for CNI Lifecycle Management and Reporting
* Cisco Data Center Fabrics automation for 5G cloud-native workloads
    * Cisco ACI automation for Pods secondary interfaces VLAN stitching
    * VLAN pools management
    * Support for additional network CNI(as a chained secondary CNI) : SR-IOV, MACVLAN, IPVLAN, Bridge, OVS
    * L2 design for fabric
    * Dynamically create BD/EPG and port attachments to the relevant Openshift nodes
    * External router attachment


## 3. Cisco Network Operator for Telco 5G cloud-native fabric orchestration

Openshift nodes are installed with the primary OVN Kubernetes CNI and Network Operator CNI chained under multus. CNI chaining allows to insert Network Operator to the chain of the network operations and influence the pod lifecycle, ensure the network connectivity is present before the CNI informs kubelet and cri-o container runtime that network plumbing is successful.
Currently CNO has been validated with Openshift 4.12 running on baremetal servers.

The objective for Cisco Network Operator in the present phase of development is to manage network attachment for secondary interfaces, set up required configuration in Cisco ACI fabric, and operate as an Operator to consistently uphold the configuration's intent.

### 3.1 Cisco ACI and Openshift node physical connectivity for CNO orchestration

| ![CNO for CNF physical connectivity](diagrams/CNO-CNF-aci-physical-topology.png) |
|:--:|
| *Openshift node physical connectivity to ACI* |

In this topology, Openshift nodes can connect to ACI Leafs using multiple interfaces. Each pair of interfaces serves a different purpose:

1. The primary POD network is managed by Openshift OVN-Kubernetes CNI and uses interfaces aggregated in the logical bond0 interface.
2. The MACVLAN CNI plugin utilizes interfaces aggregated in the bond1 logical interface.
3. The last pair of interfaces is not aggregated, but both can be used for SR-IOV Physical Functions, which will be subsequently virtualized into Virtual Functions.

### 3.2 Cisco ACI and Openshift cluster workloads logical network design

Network Operator is deployed on the Openshift cluster, and it is responsible for orchestrating the ACI fabric configuration and influencing the pod scheduling process based on network readiness, utilizing CNI chaining technology.

For each network segment, there is a corresponding BD (Bridge Domain) and EPG (End Point Group) in ACI.

Explicit physical interface and VLAN attachments are made to the relevant BD and EPG in ACI.

The BDs are configured as L2, so it is expected to connect an external router to each EPG to provide a gateway. CNO has implemented automatic attachment of the external router interface to the EPG based on a configurable mapping.

The attachment of pods to the Network Segment is accomplished through the use of a Multus Custom Resource called NetworkAttachmentDefinition (short: NAD).

| ![](diagrams/CNO-CNF-aci-logical-topology.png) |
|:--:|
| *ACI logical network design* |

## 4 Network Operator software architecture

Network Operator can be installed after deploying an Openshift cluster with the primary OVN-Kubernetes CNI.

To install Network Operator, you can use the [acc-provision](https://pypi.org/project/acc-provision/6.0.3.3/) tool, which will generate Kubernetes manifests and prepares Cisco ACI day-0 configuration. 

Optionally, you can use acc-provision tool to pre-provision Cisco ACI networking for the Openshift cluster connectivity for primary CNI.

Network Operator runs as a pods in the Openshift cluster inside the `aci-containers-system` namespace. Network Operator consists of following components:

* One pod aci-containers-controller deployment. It's main functions are:
  * subscribe to APIC selected objects and push configuration to APIC
  * Read the `nodenetworkstates.nmstate.io` CRD to discovers secondary connections using LLDP
* aci-container—host daemonset running on every node. 
  * it consists of 1 container - hostAgent which has following functions:
    * watch for NetworkAttachmentDefinitions
    * watch for Pods with additional network attachments
    * generates NodeFabricNetworkAttachment CR and other supplemental CRs explained later.

![Network Operator  software architecture](diagrams/netop-cni-architecture.png)

## 5 Network Operator installation

### 5.1 Network Operator installation pre-requisites

Before installation of the chained CNI to automate ACI fabric stitching for secondary interfaces, the following pre-requisites must be met:

1.	Install Openshift on Baremetal (validated installation methods: assisted-installer or UPI with PXE boot). Validated Openshift version: 4.12
2.	Install the following operators:
    * Multus (Openshift installation enables by default installation of multus)
    * SRIOV Operator
    * NMstate Operator
    * Ensure that LLDP is disabled on the NIC firmware
      * i.e. for Intel X710 `ethtool --set-priv-flags <eth_name> disable-fw-lldp on`
      * Note: This configuration should be implemented with a `machineconfig` otherwise will not persist between reboots
    * Enable LLDP on the node uplink interfaces designed for secondary interfaces.
      * i.e. using [nmstate](https://nmstate.io/features/lldp.html) operator


### 5.2. Network Operator installation

1. Install acc-provision on a host that has access to APIC.

    `pip install acc-provision==6.0.3.3`

2.  Prepare a YAML file named "acc-provision-input.yaml" that includes initial information about the environment, such as the APIC IP address, Tenant, VRF, CNI image registry information, and any additional parameters specified in the document. Please refer to the example "acc-provision-input-config.yaml" file for reference.

```yaml
aci_config:
  system_id: ocpbm3                   # Unique cluster name, if the Tenant is not specified this is also the tenant name
  tenant:
    name: ocpbm3                      # Add pre_existing_tenant name if it's manually created on the APIC
  apic_hosts:                         # List of APIC hosts to connect for APIC API
    - 10.0.0.1
  aep: ocp-bm-3                       # The AEP for primary CNI used by this cluster
  physical_domain:
    domain: ocp-bm-3-ovn              # physical domain name used for primary interfaces (will be attached to AAEP provided).
  vrf:                                # This VRF used for primary and secondary BDs
    name: ocp-bm-3-vrf
    tenant: common                    # Tenant where VRF is defined
  l3out:
    name: ocp3-extnet                 # Used to attach contract between primary CNI network and external world
    external_networks:
    - ocpbm3-extnet-epg               # Used for external contracts
  secondary_aep: ocp-bm-3-multus      # AAEP for secondary CNI interfaces

net_config:
  # node_subnet: 10.10.0.1/24         # Subnet to use for nodes IGNORED SINCE WE SET true for skip_node_network_provisioning
  # kubeapi_vlan: 502                 # The VLAN used by the physdom for nodes IGNORED SINCE WE SET true for skip_node_network_provisioning
 
chained_cni_config: 
  secondary_interface_chaining: true   # enable chained config
  use_global_scope_vlan: true              # use unique VLANs per leaf switch.
  skip_node_network_provisioning: true     # if true, Cisco CNI do not provision EPG/BD for node network (must be provisioned before Openshift cluster will be installed).
  vlans_file: "nad_vlan_map_input.csv"     # path to the CSV file with the VLAN information.
  # primary_interface_chaining: false      # (optional) enable CNI chaining for primary CNI – Network Operator will check connectivity to gateway prior allowing Pod to start. Currently not supported by Red Hat.
  # primary_cni_path: "/mnt/cni-conf/cni/net.d/10-ovn-kubernetes.conf” # (optional) if specified, primary CNI will be chained as well – this is not required by current use-case.
  secondary_vlans: [101,102,103,104,201]   # (optional) definite list of all vlans that should be populated in VLAN Pool for secondary intefaces

registry:                                  # Registry information 
  image_prefix: quay.io/noiro
  aci_containers_host_version: 6.0.4.1.81c2369.z       # for production use GA image tag 6.0.3.3.81c2369   
  aci_containers_controller_version: 6.0.4.1.81c2369.z # for production use GA image tag 6.0.3.3.81c2369
```

3. Specific parameters for Network Operator in chained mode:

You can create new Tenant or use pre-existing one:
```yaml
aci_config:
  system_id: ocpbm3        # Unique cluster name, if the Tenant is not specified this is also the tenant name
  tenant:
    name: ocpbm3           # Add pre_existing_tenant name if it's manually created on the APIC
```
* 2 AAEPs should be created before running acc-provision (prerequisite):
  * `aci_config.aep` - AAEP for primary CNI interfaces
  * `aci_config.secondary_aep` - AAEP for secondary CNI interface
* 2 Physical Domains (can be provided or if not specified, will be created):
  * `aci_config.physical_domain.domain` - Phys Dom for primary CNI interfaces (should exists)
  * Phys Dom for secondary CNI interfaces – automatically created with the name: `<system_id>-secondary`
* `chained_cni_config.secondary_interface_chaining` – enables CNI chaining with MACVLAN and SRIOV CNI plugins for secondary interfaces.
* `chained_cni_config.skip_node_network_provisioning` - Network provisioning for primary CNI (BD/EPG is specific VRF / Tenant, contract to provided L3out).
* `chained_cni_config.use_global_scope_vlan` – for a given VLAN one EPG is created even if multiple Network Attachment Definition refers to the same VLAN. If False, for each NAD using the same VLAN unique EPG will be created. 
* `chained_cni_config.vlans_file` – read CSV file to load VLAN id and create NadVlanMap Custom Resource at Day-0. This feature has been developed to meet specific Customer CNF operation guidance.
* `chained_cni_config.secondary_vlans` – List of VLANs used by CNO to provision VLAN Pool attached to <system_id>-secondary physical domain. This domain is attached to the EPGs created by CNO. If ip_sheet is specified, the vlan pool can be populated from the excel sheet.

4. Run acc-provision on the host that has access to APIC. Script will generate output file. 

   __Warning__: This steps will push configuration to APIC

```bash
acc-provision -a -c acc-provision-config.yml -u <apic_user> -p <apic_password> -f openshift-sdn-ovn-baremetal -o acc_deployment.yaml
```
5. Apply the output file to the openshift cluster

```bash
oc apply -f acc_deployment.yaml
```

6. Once manifest applied to the Openshift cluster, you should see following resources:

```
[root@ocp-3-orch macvlan-2]# oc get pods -n aci-containers-system -o wide
NAME                                         READY   STATUS    RESTARTS   AGE     IP             NODE                         NOMINATED NODE   READINESS GATES
aci-containers-controller-57c766b7cd-ljfp6   1/1     Running   0          3h39m   192.168.23.7   worker2.ocpbm3.noiro.local   <none>           <none>
aci-containers-host-4mwk8                    1/1     Running   0          4h9m    192.168.23.3   master1.ocpbm3.noiro.local   <none>           <none>
aci-containers-host-c42nq                    1/1     Running   0          4h9m    192.168.23.7   worker2.ocpbm3.noiro.local   <none>           <none>
aci-containers-host-cpl98                    1/1     Running   0          4h8m    192.168.23.4   master2.ocpbm3.noiro.local   <none>           <none>
aci-containers-host-s4trc                    1/1     Running   0          4h9m    192.168.23.6   worker1.ocpbm3.noiro.local   <none>           <none>
aci-containers-host-x2q6w                    1/1     Running   0          4h9m    192.168.23.5   master3.ocpbm3.noiro.local   <none>           <none>
```
Installed CRDs:

```
[root@ocp-3-orch macvlan-2]# oc get crd | grep aci.fabricattachment
nadvlanmaps.aci.fabricattachment                                  2023-08-09T12:21:05Z
nodefabricnetworkattachments.aci.fabricattachment                 2023-07-24T22:28:18Z
staticfabricnetworkattachments.aci.fabricattachment               2023-08-25T09:32:49Z
fabricvlanpool.aci.fabricattachment                               2023-09-14T20:13:23Z
```
Depending on configuration, if you provided nad-vlan-file.csv to the input, you should see `nadvlanmap` Custom Resource created and fabricvlanpool. 

Default FabricVlanPools Custom Resource:
```
[root@ocp-3-orch ~]# oc get fabricvlanpools.aci.fabricattachment -A
NAMESPACE               NAME            AGE
aci-containers-system   default         14d
```

If the NadVlanMap file has been provided in aci-prov-input.yaml file, you should see NadVlanMap Custom Resource created:
```
[root@ocp-3-orch ~]# oc get nadvlanmaps.aci.fabricattachment
NAME           AGE
nad-vlan-map   19d
```
## 6. Network Operator - quick start guide

**&#9432;** ___NOTE:___ _Make sure to have Multus installed, LLDP enabled on the interfaces used to attach additional networks to the pods_

### 6.1 Network Operator - orchestrate fabric configuration for SR-IOV interfaces connected to POD

1. Create `SriovNetworkNodePolicy`, refer for details how to use SR-IOV Operator in [documentation](https://github.com/openshift/sriov-network-operator).

Example: 
```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
generation: 1
  name: sriov-policy-enp216s0f0-0-15
  namespace: openshift-sriov-network-operator
spec:
  deviceType: netdevice
  isRdma: false
  needVhostNet: false
  nicSelector:
    deviceID: 158b
    pfNames:
    - enp216s0f0#0-63
    rootDevices:
    - 0000:d8:00.0
    vendor: "8086"
  nodeSelector:
    feature.node.kubernetes.io/network-sriov.capable: "true"
  numVfs: 64
  priority: 99
  resourceName: enp216s0f0
```

2. Create `NetworkAttachmentDefinition` (NAD)
```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  annotations:
    k8s.v1.cni.cncf.io/resourceName: openshift.io/enp216s0f0
name: sriov-net1
  namespace: default
spec:
  config: |
{
    "cniVersion": "0.3.1",
    "name": "sriov-net1",
    "plugins": [
        {
            "name": "sriov-net1",
            "cniVersion": "0.3.1",
            "type": "sriov",
            "vlan": 603,
            "trust": "on",
            "vlanQoS": 0,
            "capabilities": {
                "ips": true
            },
            "link_state": "auto",
            "ipam": {
                "type": "static",
                "addresses": [
                    {
                        "address": "192.168.128.66/24"
                    }
                ]
            }
        },
################################################
#        ADD THIS SECTION TO THE NAD:          #
################################################
        { 
            "supportedVersions": [
                "0.3.0",
                "0.3.1",
                "0.4.0"
            ],
            "type": "netop-cni",
            "chaining-mode": true,
            "log-level": "debug",
            "log-file": "/var/log/netopcni.log"
        }
###############################################
    ]
}
```
Once NAD is created Network Operator creates Bridge Domain / Endpoint Group in Cisco ACI for the VLAN specified in the NetworkAttachmentDefinition. The name of EPG and BD are hardcoded using the following schema: 
* `secondary-vlan-<vlanID>`
* `secondary-bd-vlan-<vlanID>`

| ![](diagrams/quick-start-epg-603.png)
|:--:|
| *BD/EPG created in Cisco ACI after NAD deployment* |

`aci-containers-host` watches for NAD creation and creates `NodeFabricNetworkAttachment` (NFNA) Custom Resource per NAD and per Openshift node on which the NetworkAttachmentDefinition applies. 

NFNA aggregates information about:
  * Host interface and discovered port on ACI
  * Pod reference (will be visible after scheduling pod attached to the NAD)
  * VLAN ID or multiple IDs
  * Reference to the NAD
  * Reference to the node
  * Primary CNI in the chain (this is still CNI for secondary interfaces, but it is called primary in the CNI chain)
```bash
[root@ocp-3-orch ~]# oc get nodefabricnetworkattachments.aci.fabricattachment -n aci-containers-system
NAME                                             AGE
worker1.ocpbm3.noiro.local-default-sriov-net-1   85m
worker2.ocpbm3.noiro.local-default-sriov-net-1   85m
```
```yaml

[root@ocp-3-orch ~]# oc get nodefabricnetworkattachments.aci.fabricattachment -n aci-containers-system   worker1.ocpbm3.noiro.local-default-sriov-net-1 -o yaml
apiVersion: aci.fabricattachment/v1
kind: NodeFabricNetworkAttachment
metadata:
  name: worker1.ocpbm3.noiro.local-default-sriov-net-1
  namespace: aci-containers-system
spec:
  aciTopology:
    ens1f2:
      fabricLink:
      - topology/pod-1/node-101/pathep-[eth1/41]
  encapVlan:
    encapRef:
      key: ""
      nadVlanMap: ""
    vlanList: "603"
  networkRef:
    name: sriov-net-1
    namespace: default
  nodeName: worker1.ocpbm3.noiro.local
  primaryCni: sriov
  ```

3. create a Pod, refer to the additional network attachment using annotation:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multitool-sriov-demo-1
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: sriov-net-1
  labels:
    run: multitool
spec:
  containers:
  - name: network-multitool
    image: wbitt/network-multitool
  nodeName: worker1.ocpbm3.noiro.local
```

Pod scheduled on specific worker node (worker1). Network Operator configure Static Path towards the node that runs a Pod attached to the NAD.
The interface is taken from NodeFabricNetworkAttachment Custom Resource. 


| ![Alt text](diagrams/static-port-epg.png) | 
| :--: |
| *Static Port configured under EPG* |

Network Operator compute and maintains NodeFabricNetworkAttachment resource in Kubernetes. Example of a resource after creating NAD and attach Pod to it:

```yaml
apiVersion: aci.fabricattachment/v1
kind: NodeFabricNetworkAttachment
metadata:
  name: worker1.ocpbm3.noiro.local-default-sriov-net-1
  namespace: aci-containers-system
spec:
  aciTopology:
    ens1f2:                                       
      fabricLink:
      - topology/pod-1/node-101/pathep-[eth1/41]  # Discovered fabric interface
      pods:
      - localIface: ens1f2v34                     # Pod details with VF interface
        podRef:
          name: multitool-sriov-demo-1
          namespace: default
  encapVlan:
    encapRef:
      key: ""
      nadVlanMap: ""
    vlanList: "603"                               # VLAN information
  networkRef:
    name: sriov-net-1
    namespace: default
  nodeName: worker1.ocpbm3.noiro.local            # Node for which this NFNA has been created.
  primaryCni: sriov                               # CNI plugin used
```

### 6.2 Network Operator - orchestrate fabric configuration for MACVLAN interfaces connected to Pod

Similar to SR-IOV interfaces, additional interfaces managed by MACVLAN CNI are supported by Network Operator. It will configure Cisco ACI fabric with relevant Static Ports under EPG. In case of bonded uplink, Network Operator discovers port-channel members and configures static port under EPG referring to the Virtual Port-Channel or Port-Channel Interface Group Policy.

In order to specify VLAN ID in the NetworkAttachmentDefinition directly, configure subinterface on the uplink that will be used by MACVLAN, and refer subinterface in NetworkAttachmentDefinition.

Example:
1. Create subinterface on bond1 port-channel interface on worker2 using NMState Operator.

```yaml
apiVersion: nmstate.io/v1beta1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: bond1.604-worker2
spec:
  nodeSelector:
    kubernetes.io/hostname: worker2.ocpbm3.noiro.local
  desiredState:
    interfaces:
    - name: bond1.604
      type: vlan
      state: up
      vlan:
        base-iface: bond1
        id: 604
```

2. Create NetworkAttachmentDefinition

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-net2-bond1-604
  namespace: default
spec:
  config: |
{
    "cniVersion": "0.3.1",
    "name": "macvlan-net2-bond1-604",
    "plugins": [
        {
            "cniVersion": "0.3.1",
            "name": "macvlan-net2-bond1-604",
            "type": "macvlan",
            "mode": "private",
            "master": "bond1.604",
            "ipam": {
                "type": "whereabouts",
                "range": "192.168.100.0/24",
                "exclude": [
                    "192.168.100.0/32",
                    "192.168.100.1/32",
                    "192.168.100.254/32"
                ]
            }
        },
        {
            "supportedVersions": [
                "0.3.0",
                "0.3.1",
                "0.4.0"
            ],
            "type": "netop-cni",
            "chaining-mode": true,
            "log-level": "debug",
            "log-file": "/var/log/netopcni.log"
        }
    ]
}
```

Network Operator reacts to the NetworkAttachmentDefinition create event, and will create NodeFabricNetworkAttachment for that specific NAD - again, one per node where the NAD is applicable. Network Operator will also push BD and EPG for VLAN-604 to the APIC.

3. Create Pod manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multitool-macvlan-net2-pod2
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-net2-bond1-604
  labels:
    cni: macvlan
spec:
  containers:
  - name: network-multitool
    image: wbitt/network-multitool
  nodeName: worker2.ocpbm3.noiro.local
```
Network Operator will create a NodeFabricNetworkAttachment resource based on the NAD `macvlan-net2-bond1-604`

```yaml
apiVersion: aci.fabricattachment/v1
kind: NodeFabricNetworkAttachment
metadata:
  name: worker2.ocpbm3.noiro.local-default-macvlan-net2-bond1-604
  namespace: aci-containers-system
spec:
  aciTopology:
    bond1:                                         # Bond host interface reference
      fabricLink:
      - topology/pod-1/node-101/pathep-[eth1/43]   # Discovered individual interfaces through LLDP
      - topology/pod-1/node-102/pathep-[eth1/43]   # Discovered individual interfaces through LLDP
      pods:
      - localIface: net1
        podRef:
          name: multitool-macvlan-net2-pod2
          namespace: default
  encapVlan:
    encapRef:
      key: ""
      nadVlanMap: ""
    vlanList: "604"
  networkRef:
    name: macvlan-net2-bond1-604
    namespace: default
  nodeName: worker2.ocpbm3.noiro.local
  primaryCni: macvlan
```

Based on the Ethernet ports discovered by Network Operator, The Virtual Port Channel has been discovered and added to the EPG.

| ![Alt text](diagrams/static-port-epg.png) |
|:--:|
| *Static Port for VPC interface automatically discovered* |

### 6.3 Provisioning multiple VLANs within the same NetworkAttachmentDefiniton

NetworkAttachmentDefinition allows to specify VLAN ID in multiple ways, depending on the defined plugin CNI. For SR-IOV CNI, you can specify vlan ID in the plugins.vlan field - this will configure VLAN ID for the VF on the NIC card directly and NIC card will encapsulate traffic from the attached Pod on the wire.

In many 5G CNF use-cases, encapsulation is done on the Pod itself, and VF should be treated as a trunk allowing list of VLANs. 

Network Operator allows that configuration through annotation or reference to the resource "nad-vlan-map". The last one has been developed for specific use-case and is described in [Chapter 8.2.1](#821-nadvlanmap-custom-resource).

The standard way of configuring VLAN list for the NAD is done through annotations. Network Operator provisions appropriate network segments on Cisco ACI fabric - for each VLAN pair of BD/EPG.

Example: 
```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: sriov-net1
  namespace: default
  annotations:
    netop-cni.cisco.com/vlans: ‘[100,103,106-108]’ 
    k8s.v1.cni.cncf.io/resourceName: openshift.io/<resourceName>
```

Above configurartion results in creating 5 BD/EPG in ACI for vlan: 100, 103, 106, 107, 108. When Pod will be attached to this NAD, the static binding will be created for each EPG created for each vlan from this list. Since it is unknown which VLAN a Pod will choose to encapsulate traffic, all vlans are allowed and provisioned for any pod attached to this NAD.


### 6.4 Orchestrate fabric configuration for IPVLAN interfaces connected to Pod

Additional interfaces managed by IPVLAN CNI are supported by Network Operator. The uplink port connecting to the fabric can be either an untagged port or a subinterface with a vlan. Since the model of segmentation with IPVLAN is to use a different subnet for a different network, pod interface is always untagged. Furthermore, only one vlan should be selected for the NAD to be used as the global vlan for all the networks based on this NAD, and an error will be displayed on the subsequent NFNA and pods will not comeup, if more than one vlan is selected for this NAD. This vlan although, not visible to the pods using this NAD, will be seen by the fabric in case of tagged uplink port or implicitly used in case of untagged uplink port.

Example:
1. Create subinterface on bond1 port-channel interface on worker2 using NMState Operator.

```yaml
apiVersion: nmstate.io/v1beta1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: bond1.103-worker2
spec:
  nodeSelector:
    kubernetes.io/hostname: worker2.ocpbm1.noiro.local
  desiredState:
    interfaces:
    - name: bond1.103
      type: vlan
      state: up
      vlan:
        base-iface: bond1
        id: 103
```
2. Create NetworkAttachmentDefinition

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  creationTimestamp: "2024-01-26T19:26:53Z"
  generation: 1
  name: ipvlan-net2
  namespace: default
  resourceVersion: "197623206"
  uid: c316c4ea-7089-445f-9dc8-638d13195609
spec:
  config: |-
  '{"cniVersion": "0.3.1", 
    "name": "ipvlan-net2", 
    "plugins":[{"cniVersion": "0.3.1", 
                "name": "ipvlan-net2", 
                "type": "ipvlan",
                "mode": "l2",
                "master": "bond1.103",
                "ipam": { "type": "host-local", 
                          "subnet": "10.3.3.0/24"}
               },
              {
               "supportedVersions": [
                "0.3.0",
                "0.3.1",
                "0.4.0"
            ],
            "type": "netop-cni",
            "chaining-mode": true,
            "log-level": "debug",
            "log-file": "/var/log/netopcni.log"
        }]}'
```
Network Operator reacts to the NetworkAttachmentDefinition create event, and will create NodeFabricNetworkAttachment for that specific NAD - again, one per node where the NAD is applicable. Network Operator will also push BD and EPG for VLAN-103 to the APIC.

3. Create Pod manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multitool-ipvlan-net2-pod1
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: ipvlan-net2
  labels:
    cni: ipvlan
spec:
  containers:
  - name: network-multitool
    image: wbitt/network-multitool
  nodeName: worker2.ocpbm1.noiro.local
```

```yaml
apiVersion: aci.fabricattachment/v1
kind: NodeFabricNetworkAttachment
metadata:
  name: worker2.ocpbm1.noiro.local-default-ipvlan-net2
  namespace: aci-containers-system
spec:
  aciTopology:
    bond1:
      fabricLink:
      - topology/pod-1/node-101/pathep-[eth1/21]
      - topology/pod-1/node-102/pathep-[eth1/21]
    pods:
      - localIface: net1
        podRef:
          name: multitool-ipvlan-net2-pod1
          namespace: default
  encapVlan:
    encapRef:
      key: ""
      nadVlanMap: ""
    mode: Trunk
    vlanList: '[103]'
  networkRef:
    name: ipvlan-net2
    namespace: default
  nodeName: worker2.ocpbm1.noiro.local
  primaryCni: ipvlan
status:
  state: Complete
```
NFNA created by operator will look like above. The encapvlan-mode is Trunk in this case, because the uplink is tagged with the vlan 103 and packets to the uplink will use vlan 103. In case the uplink is untagged, the mode will say Access(Untagged). L3 mode with IPVLAN is not supported at this time.

### 6.5 Orchestrate fabric configuration for bridge(linux-bridge) interfaces connected to Pod

Additional interfaces managed by bridge CNI are supported by Network Operator. In this case the uplink port is expected to be connected to the bridge as a bridge-port. It is recommended that the uplink port be untagged, in order to allow for pod interface trunking and hence multiple networks off a single uplink. If a single uplink subintf is used in the NAD, then only a single network will be possible. When the uplink is added as a bridge port, CNO will still listen to the LLDP messages on the uplink and do automatic provisioning.

Example:
1. Create a bridge and add the uplink port to the bridge on worker1 using NMState Operator.

```yaml
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: bridge-bond1
spec:
  nodeSelector:
    kubernetes.io/hostname: worker1.ocpbm1.noiro.local
  desiredState:
    interfaces:
    - name: bridge-net1
      description: Linux bridge with bond1 as a port
      type: linux-bridge
      state: up
      bridge:
        port:
        - name: bond1
```

2. Create NetworkAttachmentDefinition. 

Ensure that the list of vlans that you want to trunk is present in the definition. In case you want to use this definition with kubevirt, ensure that kubevirt resource annotation is present. Openshift uses cnv-bridge as the type of CNI and that should work as well.

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: bridge-net1
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/bridge-net1
spec:
  config: |-
   '{"cniVersion": "0.3.1",
     "name": "bridge-net1", 
     "plugins":[{  
         "cniVersion": "0.3.1",
         "name": "bridge-net1",
         "type": "bridge",
         "bridge": "bridge-net1",
         "vlanTrunk": [{ "id": 105 },{ "minID": 102, "maxID": 104 }],
       },
       {
          "supportedVersions": [
              "0.3.0",
              "0.3.1",
              "0.4.0"
          ],
          "type": "netop-cni",
          "chaining-mode": true,
        }
       ]}'
  ```
Network Operator reacts to the NetworkAttachmentDefinition create event, and will create NodeFabricNetworkAttachment for that specific NAD - again, one per node where the NAD is applicable. Network Operator will also push BD and EPG for VLAN-102 to 105 to the APIC.

3. Create Pod manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multitool-bridge-net1-pod1
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: bridge-net1
  labels:
    cni: bridge
spec:
  containers:
  - name: network-multitool
    image: wbitt/network-multitool
  nodeName: worker1.ocpbm1.noiro.local
```

```yaml
apiVersion: aci.fabricattachment/v1
kind: NodeFabricNetworkAttachment
metadata:
  name: worker1.ocpbm1.noiro.local-default-bridge-net1
  namespace: aci-containers-system
spec:
  aciTopology:
    bridge-net1:
      fabricLink:
      - topology/pod-1/node-101/pathep-[eth1/21]
      - topology/pod-1/node-102/pathep-[eth1/21]
    pods:
      - localIface: net1
        podRef:
          name: multitool-bridge-net1-pod1
          namespace: default
  encapVlan:
    encapRef:
      key: ""
      nadVlanMap: ""
    mode: Trunk
    vlanList: '[102-105]'
  networkRef:
    name: bridge-net1
    namespace: default
  nodeName: worker1.ocpbm1.noiro.local
  primaryCni: bridge
status:
  state: Complete
```
L3 mode with bridge cni is not supported at this time.

### 6.6 Orchestrate fabric configuration for ovs bridge interfaces connected to Pod

Additional interfaces managed by ovs CNI are supported by Network Operator. In this case the uplink port is expected to be connected to the ovs bridge as a bridge-port. It is recommended that the uplink port be untagged, in order to allow for pod interface trunking and hence multiple networks off a single uplink. If a single uplink subintf is used in the NAD, then only a single network will be possible. When the uplink is added as a bridge port, CNO will still listen to the LLDP messages on the uplink and do automatic provisioning.
OVS CNI support is added to the aci-containers-host-ovscni image only, so ensure that image is used if using directly or ```enable_ovs_cni_support: true``` under ```chained_cni_config``` for acc-provision.

Example:
1. Use platform specific methods to install and configure OpenVswitch on the hosts and configure ovs-bridge and ports.
e.g. On Openshift platform, use node network configuration policy.
```
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: ovs-bridge
spec:
  desiredState:
    interfaces:
    - name: br0
      type: ovs-bridge
      state: up
      bridge:
        options:
          stp: false
        port:
        - name: bond1
```
If no platform specific methods are available, install openvswitch from source or package, create an ovs bridge and add the uplink port to the bridge using ovs-vsctl commands.
```
ovs-vsctl add-br br0
ovs-vsctl add-port br0 bond1
```

2. Create NetworkAttachmentDefinition. 

Ensure that the name of the bridge corresponds to the one created and list of vlans that you want to trunk is present in the definition. 

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: ovs-br0
  namespace: default
spec:
  config: |-
  '{"cniVersion": "0.3.1",
    "name": "ovs-br0",
    "plugins":[{
      "cniVersion": "0.3.1", "name": "ovs-br0", "type": "ovs", "bridge": "br0", 
      "vlanTrunk": [{ "minID": 3804, "maxID": 3805 }]}, 
      { "supportedVersions": [ "0.3.0", "0.3.1", "0.4.0" ], "type": "netop-cni", "chaining-mode": true}]}'
```

Network Operator reacts to the NetworkAttachmentDefinition create event, and will create NodeFabricNetworkAttachment for that specific NAD - again, one per node where the NAD is applicable. Network Operator will also push BD and EPG for VLAN-3804 to 3805 to the APIC.

3. Create Pod manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multitool-ovs-br0-pod1
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: ovs-br0
  labels:
spec:
  containers:
  - name: network-multitool
    image: wbitt/network-multitool
  nodeName: worker1.ocpbm3.noiro.local
```

```yaml
apiVersion: aci.fabricattachment/v1
kind: NodeFabricNetworkAttachment
metadata:
  name: worker1.ocpbm3.noiro.local-default-ovs-br0
  namespace: aci-containers-system
spec:
  aciTopology:
    br0:
      fabricLink:
      - topology/pod-1/node-101/pathep-[eth1/39]
    pods:
      - localIface: net1
        podRef:
          name: multitool-ovs-br0-pod1
          namespace: default
  encapVlan:
    encapRef:
      key: ""
      nadVlanMap: ""
    vlanList: '[3804-3805]'
  networkRef:
    name: ovs-br0
    namespace: default
  nodeName: worker1.ocpbm3.noiro.local
  primaryCni: ovs
status:
  state: Complete
```

## 7.  Webhook for adding CNO into the NAD CNI chain

To avoid user configuration to add CNO to the end of NAD CNI chain, one can use the CNO webhook. 

### 7.1 Provisioning webhook for CNO insertion into NAD CNI chain.

Use the following configuration in acc-provision-input.yaml to allow running the CNI webhook to insert CNO into the NAD CNI chain. If using with openshift- local_cert_manager is not required.
If require_annotation_for_nad_mutation_webhook is not set to true, then all NADs in the cluster will have CNO inserted at the end of the chain. Enabling this option, will ensure that CNO is inserted to a NAD, only when the annotation ```netop-cni.cisco.com/auto-chain-cni="true"``` is present on the NAD.

```yaml
chained_cni_config:
  secondary_interface_chaining: true
  auto_insertion_for_nad: true
  local_cert_manager_enabled: true
  require_annotation_for_nad_mutation_webhook: true
```

## 8. Custom features for Ericsson Packet Core support

### 8.1 Custom requirements

Ericsson 5G Packet Core application deployment on Openshift includes Network Attachment Definitions to connect pods to additional networks based on MACVLAN and SR-IOV. Here is a list of custom requirements for NAD specification:
(1) NAD manifest for SR-IOV has VLAN ID = 0, while NAD for MACVLAN refers to the parent bond interface, without subinterface. 
(2) NetworkAttachmentDefinition doesn't consists of VLAN ID, it is assumed that CNF PODs will trunk multiple VLANs. 
(3) Unique network segment (VLAN) will be shared across NetworkAttachmentDefinitions to connect CNF workloads using SR-IOV and MACVLAN CNI's. 
(4) Mapping of VLANs to NADs has been provided in form of excel spreadsheet, which is assumed to be a rather static, and part of DAY-1 configuration.
(5) Currently validated design assumes that Cisco ACI fabric acts as a Layer-2 switching network, routing between network segments will be configured centrally on the attached external router.

### 8.2 Network Operator implementation

Network Operator has been designed to address Ericsson requirements, and allow managing VLAN pools and maps them to the NetworkAttachmentDefinition.

#### 8.2.1 NadVlanMap Custom Resource

The Ericsson Packet Core application performs VLAN encapsulation inside the Pod network namespace, hence NetworkAttachmentDefinition does not have a VLAN ID specified. NAD only refers to the parent uplink interface (in case of MAC VLAN it will be parent interface) or refer to the SRIOV Network Node Policy name and VLAN ID = 0. 

Network Operator exposes Custom Resource named NadVlanMap, which provides mapping between NetworkAttachmentDefinition (NAD) name and possible VLAN ID's that are assumed to be used by PODs connected to the NAD.
Example manifest of the NadVlanMap:

```yaml
apiVersion: aci.fabricattachment/v1
kind: NadVlanMap
metadata:
  name: nad-vlan-map               # This is singleton resource, must use exact "nad-vlan-map" name
  namespace: aci-containers-system
spec:
  nadVlanMapping:
    namespace-A/nad-1:       # namespace (fullname match) / NADs prefix name (first letters match)
      - label: app-1         # Label is not used for matching NAD to VLAN.
        vlans: "101"
      - label: app-2
        vlans: "102"
      - label: app-3
        vlans: "103"
    namespace-A/nad-2:
      - label: app-1
        vlans: "101"         # the same vlans can be used by multiple NADs
      - label: app-2
        vlans: "102"
      - label: app-4
        vlans: "104"
    namespace-B/nad-1:       # NetworkAttachmentDefinition is a namespaced resource and can duplicate across namespaces
      - label: app-1
        vlans: "201"
```

NadVlanMapping provides list of VLANs to be assigned to the NAD. NAD key is using pattern: `<namespace>/<nad_name_prefix>`
**&#9432;** ___NOTE:___ _NadVlanMap resource is a singleton (one per cluster) resource and its name must be: “nad-vlan-map”_
The "label" attribute is not used for matching a specific VLAN to a Network Attachment Definition (NAD)/Pod. Instead, it is utilized by the NetworkFabricNetworkConfiguration, which will be explained in detail later in the document.

When NetworkAttachmentDefinition resource is created, Network Operator finds matching namespace / NAD name in the NadVlanMap, it creates as many BD/EPG pairs as the number of VLANs in the list for matched namespace/NAD prefix.

If you look at the above NadVlanMap, and you would create NAD with the name: `nad-1` in the namespace `namespace-A`, Network Operator will create BD and EPGs with the following names:
- ACI Bridge Domains (BD):
  - secondary-bd-vlan-101
  - secondary-bd-vlan-102
  - secondary-bd-vlan-103
- ACI Endpoint Groups (EPG):
  - secondary-vlan-101
  - secondary-vlan-102
  - secondary-vlan-103

Network Operator creates NodeFabricNetworkAttachment resource - one per NAD and per node. It will incorporate information about allowed VLAN list and reference to the matching entry in NadVlanMap resources. 

Example:

```yaml
apiVersion: aci.fabricattachment/v1
kind: NodeFabricNetworkAttachment
metadata:
  name: worker1.ocpbm3.noiro.local-namespace-A-nad-1
  namespace: aci-containers-system
spec:
  aciTopology:
    bond1:
      fabricLink:
      - topology/pod-1/node-101/pathep-[eth1/40]
      - topology/pod-1/node-102/pathep-[eth1/40]
  encapVlan:
    encapRef:
      key: namespace-A/nad-1                             # found matching in NadVlanMap
      nadVlanMap: aci-containers-system/nad-vlan-map     # reference to the NadVlanMap resource
    vlanList: '[101,102,103]'                            # list of VLANs defined in the NadVlanMap matching the Namespace/NAD prefix
  networkRef:
    name: nad-1
    namespace: namespace-A
  nodeName: worker1.ocpbm3.noiro.local
  primaryCni: macvlan
```

When Pod is created, it is expected that the POD will do VLAN tagging its `netX` interface. Consequently, the provisioning of static paths towards the relevant worker node is carried out across all Endpoint Groups (EPGs) that map the VLAN to that NetworkAttachmentDefinition (NAD).

When multiple NetworkAttachmentDefinitions refers to the same VLAN, and the Network Operator has been provisioned with the `chained_cni_config.use_global_scope_vlan` set to `true`, then Bridge Domain / Endpoint Group will be created only once, and every consecutive NAD that refer to the same VLAN will use exisiting BD EPG for attaching static port. 

Example: 

Network Operator creates NodeFabricNetworkAttachment for NetworkAttachmentDefinition with the name `nad-1` in namepsace `namespace-A`. In addition Pod has been created and attached to the NAD using annotation `"k8s.v1.cni.cncf.io/networks: nad-1"`
```yaml
apiVersion: aci.fabricattachment/v1
kind: NodeFabricNetworkAttachment
metadata:
  name: worker1.ocpbm3.noiro.local-namespace-A-nad-1
  namespace: aci-containers-system
spec:
  aciTopology:
    bond1:
      fabricLink:
      - topology/pod-1/node-101/pathep-[eth1/40]
      - topology/pod-1/node-102/pathep-[eth1/40]
      pods:
      - localIface: net1                                 
        podRef:                                          # Reference to the pod attached to the NAD.
          name: app-1
          namespace: namespace-A
  encapVlan:
    encapRef:
      key: namespace-A/nad-1                             # found matching in NadVlanMap
      nadVlanMap: aci-containers-system/nad-vlan-map     # reference to the NadVlanMap resource
    vlanList: '[101,102,103]'                            # list of VLANs defined in the NadVlanMap matching the Namespace/NAD prefix
  networkRef:
    name: nad-1
    namespace: namespace-A
  nodeName: worker1.ocpbm3.noiro.local
  primaryCni: macvlan
```
Another NAD - `nad-2` in the same namespace has been created, and Pod attached to it. For those resources, Network Operator creates another `NodeFabricNetworkAttachment` resource. 

```yaml
apiVersion: aci.fabricattachment/v1
kind: NodeFabricNetworkAttachment
metadata:
  name: worker1.ocpbm3.noiro.local-namespace-A-nad-2
  namespace: aci-containers-system
spec:
  aciTopology:
    ens1f2:                                        
      fabricLink:
      - topology/pod-1/node-101/pathep-[eth1/41]  # Discovered fabric interface
      pods:
      - localIface: ens1f2v34                     # Pod details with VF interface
        podRef:
          name: app-1
          namespace: namespace-A
  encapVlan:
    encapRef:
      key: namespace-A/nad-2                             # found matching in NadVlanMap
      nadVlanMap: aci-containers-system/nad-vlan-map     # reference to the NadVlanMap resource
    vlanList: '[101,102,104]'                            # list of VLANs defined in the NadVlanMap matching the Namespace/NAD prefix (nad-2)
  networkRef:
    name: nad-2
    namespace: namespace-A
  nodeName: worker1.ocpbm3.noiro.local
  primaryCni: sriov
```

Network Operator will create only one new BD/EPG for vlan 104, since it does not overlap with any previously created BD/EPGs. 
For the Pod, Network Operator will create Static Port in two existing EPGs vlan-101 and vlan-102 refering to the interface Eth1/41 on Leaf 101 and add vlan encapsulation respectively for each EPG. 

#### 8.2.2 FabricVlanPool custom reosource

You can maintain list of VLAN mapping to NADs by editing NadVlanMaps resource, and add/remove vlans, NAD prefixes. This will update the list of EPGs that will be created when new NAD will be created, however, it does not update the VLAN Pool `“<systemID>-secondary”` in Fabric Access Policies. 

After Network Operator installation, acc-provision generates `FabricVlanPool` resource named `“default”` in `aci-containers-system` namespace. It will consists all VLANs specified in the net_config.secondary_vlans list (one of the acc-provision-input parameters), or if the secondary_vlan list is not provided, the VLAN list will be derived from input `nad_vlan_map_input.csv` file.
You can create additional FabricVlanPool CRDs in another namespace and add more vlans to the VLAN Pool in Fabric access policies. 
The VLAN Pool will be always union of all users defined FabricVlanPools and the default one. 

```yaml
apiVersion: aci.fabricattachment/v1
kind: FabricVlanPool
metadata:
  name: default                      # default FabricVlanPool
  namespace: aci-containers-system
spec:
  vlans:
  - "101-104"
  - "201"
```

You can add more vlans by editing the `default` FabricVlanPool resource, or by creating your own in the namespace where your NAD definition will be created.
To add new vlans in FabricVlanPool resource without affecting traffic, always add new vlans in a separate range in the fabricvlanpool CR, if directly editing the CR(If using nad-vlan-map, acc-provision addresses this: see section on NAD VLAN map case).
If each vlan in the new range is expected to removed at some point of time, please add that vlan in a range by itself(eg: 600-600 or comma separated <>,600). Otherwise removal of any vlan in the range would affect traffic in the other vlans momentarily(since range cannot be modified and has to be deleted and recreated to remove a specific vlan in the range).

**&#9432;** ___NOTE:___ _when no match will be found for NAD prefix name in the NadVlanMap, Network Operator will create BD/EPGs for each VLAN defined in the FabricVlanPool `default` and FabricVlanPool in the namespace where NAD is created._ 

If multiple application owners requires Network Operator to automate Cisco ACI configuration for their own CNF applications, you can assign VLAN ranges per application owner, and create dedicated FabricVlanPools for each of them in their respective namespaces. You can skip populating `default` FabricVlanPool by not providing `chained_cni_config.secondary_vlan` list nor `chained_cni_config.vlans_file` input file, and leverage only user created `FabricVlanPool` resources.

#### 8.2.3 External router port attachment

Cisco ACI fabric acts as a layer 2 switched network for a Pod secondary interfaces. Routing to the other subnets will be performed on an external router that should be attached to the same VLAN. To automate attachment of an external gateway uplink to each EPG, the additional Custom Resource has been defined. `NetworkFabricConfiguration` is a CRD that matches label from `NadVlanMap` and AAEP where the external router is connected.

Based on above custom resource, Network Operator matches VLAN and EPG name based on label in the `NadVlanMap` and associates it with defined AAEP. There is no static port added to the EPG for the router, just all interfaces that belongs the the AAEP will get programmed a VLAN that matches the label.

**&#9432;** ___NOTE:___ _A prerequisite for this implementation is to dedicate unique AAEP for external router. CNO doesn’t do specific staticPath binding to each epg for external router, rahter it is associating EPG with AAEP in __Fabric -> Access Policies -> Policies -> Global -> Attachable Access Entity Profile__._

Example NetworkFabricConfiguration resource:

```yaml
apiVersion: aci.fabricattachment/v1
kind: NetworkFabricConfiguration
metadata:
  name: networkfabricconfiguration
  namespace: aci-containers-system
spec:
   nadVlanRefs:
   - nadVlanLabel: "app-1"   # Option 1: Match EPG based on the label defined in NadVlanMap
     aeps:
     - "router-aaep"
   vlans:
   - vlans: "101"            # Option 2: Match EPG based on explicit VLAN ID
     aeps:
     - "router-aaep"
```
VLAN can be directly specified or to a label that is defined in the NadVlanMaps CR to associate an AEP but anyway, it must match to a VLAN that is defined in the FabricVlanPool, otherwise VLAN Pool in APIC won’t have that Vlan and fault will be raised

| ![](diagrams/nfc-diagram.png) |
|:--:|
| *NetworkFabricConfiguration logical diagram* |

#### 8.2.4 Custom EPG and BD Configuration

NetworkFabricConfiguration CR can also be used to:
- provide custom names for ApplicationProfile, EPG, BD when using vlan references directly [Doesn't work for NadVlanLabel reference].
- associate a different tenant for BD, EPG.
- associate EPG with contracts as consumer/provider.
- associate subnets with BD. Note that subnet with Gateway address has to be provided.[eg. Not 10.2.0.0/24 rather 10.2.0.1/24]
- associate VRF in common or same tenant with BD. 

Assuming that the user has pre-created the following new objects mentioned in the CR in ACI.
  - Tenant (other than the one provisioned through acc-provision) and Application profile (netop-systemid) in that tenant
  - VRF
  - Contracts

Note that the additional functionality above will only be operational after the vlan is used by a NAD.
Please see the following populated CR for an example

```yaml
apiVersion: aci.fabricattachment/v1
kind: NetworkFabricConfiguration
metadata:
  name: networkfabricconfiguration
  namespace: aci-containers-system
spec:
   nadVlanRefs:
   vlans:
   - vlans: "101"            # Option 2: Match EPG based on explicit VLAN ID
     aeps:
     - "router-aaep"
     epg:
       name: custom-epg1
       applicationProfile: custom-ap1
       bd:
         name: custom-bd1
         common-tenant: false
         subnets:
         - "10.2.3.0/24"
         vrf:
           name: common-vrf1
           common-tenant: true
       contracts:
         consumer:
         - ctrct1
         provider:
         - ctrct2
```

#### 8.2.5 VLAN file ingest

Network Operator can load VLAN mapping data at the time of installation from a *.csv. Based on the information provided in the spreadsheet, NadVlanMap manifest will be generated in the acc-provision output file.

| ![Alt text](diagrams/nad_vlan_mapping_excel.png) |
|:--:|
| *VLAN spreadsheet format* |

##### 8.2.5.1 Fresh installation use case

Excel sheet has to be converted to CSV format and path to the file has to be specified in the acc-provision-input.yaml as:

```yaml
chained_cni_config.vlans_file: "nad_vlan_map_input.csv"
```
In this example, `nad_vlan_map_input.csv` has been located in the current directory from which acc-provision will be executed.

the NadVlanMap populated with the information from CSV file will be part of acc-provision-output.yaml

```bash
acc-provision -a -f openshift-sdn-ovn-baremetal -u admin -p ”password” -c acc_provision_input.yaml -o acc-provision-output.yaml
```

##### 8.2.5.2 Adding VLANs in NAD VLAN map case with no traffic disruption

Modified excel sheet has to be converted to CSV format and path to the file has to be specified in the acc-provision-input.yaml as:

```yaml
chained_cni_config.vlans_file: "nad_vlan_map_input_v2.csv"
```
In this example, `nad_vlan_map_input_v2.csv` has been located in the current directory from which acc-provision will be executed.

Also, mention old CSV (or CSV used for last provisioning) file path in argument of acc-provision command as:

```bash
acc-provision --upgrade -f openshift-sdn-ovn-baremetal -c acc_provision_input.yaml -o acc-provision-output.yaml --old-nad-vlan-map-input nad_vlan_map_input_v1.csv
```
In this example, `nad_vlan_map_input_v1.csv` is NAD VLAN map input file used in last provisioning and has been located in the current directory from which acc-provision will be executed.

`--old-nad-vlan-map-input` - Without providing old nad-vlan-map file, vlan ranges in pool will be coalesced greedily, which may affect existing traffic, because existing pools would be deleted and recreated.

## 9. Primary CNI chaining (tech-preview)

**&#9432;** ___NOTE:___ _This is tech-preview feature, not currently supported in production environments._

The CNI chaining for primary CNI (OVN-Kubernetes) has been implemented but turned off by default while Cisco is working with Red Hat to certify CNI chaining for OVN-Kubernetes.
CNI chaining for primary interface add the capability to detect network issues and prevent Pods scheduling while the networing is not ready.

You can enable OVN-Kubernetes CNI chaining with Cisco Network Operator by adding following configuration options to the acc-provision-input.yaml file:
- `chained_cni_config.primary_interface_chaining: true` - enable feature
- `primary_cni_path: "/mnt/cni-conf/cni/net.d/10-ovn-kubernetes.conf"`- indicate path to the primary CNI configuration file.
Once CNI chaining is enabled for primary interface and Network Operator deployed on the cluster, all nodes multus configuration for ovn-kubernetes located in each node filesystem: (`/run/multus/cni/net.d/10-ovn-kubernetes.conf`) will have additional entry:

```json
{
 	"name": "ovn-kubernetes",
 	"cniVersion": "0.4.0",
 	"plugins": [{
 		"cniVersion": "0.4.0",
 		"name": "ovn-kubernetes",
 		"type": "ovn-k8s-cni-overlay",
 		"ipam": {},
 		"dns": {},
 		"logFile": "/var/log/ovn-kubernetes/ovn-k8s-cni-overlay.log",
 		"logLevel": "4",
 		"logfile-maxsize": 100,
 		"logfile-maxbackups": 5,
 		"logfile-maxage": 5
  },
  {
    "_comment": "CNO CNI CHAINING INFORMATION WILL BE ADDED:"
  },
  {
 		"supportedVersions": ["0.3.0", "0.3.1", "0.4.0"],
 		"type": "netop-cni",
 		"chaining-mode": true,
 		"log-level": "debug",
 		"log-file": "/var/log/netop-agent.log"
 	}]
}
 ```

**&#9432;** ___NOTE:___ _enabling CNI chaining on primary interface require restart of multus PODs (the same applies to disabling CNI chaining)_
```bash
oc delete pod -n openshift-multus -l app=multus
```
## 10. Network Operator - orchestrate fabric for router pods using additional interfaces

This model allows pods that have additional interfaces using any of the CNIs in sections  6.1 - 6.5 to peer with the ACI Fabric.
BGP is supported as the peering protocol as this is commonly used in container functions.
Using the `NetworkFabricL3Configuration` CR, user can specify which encap needs to be orchestrated as a router pod network
that can peer with the ACI Fabric. This CR is cluster scoped. If this vlan is chosen by the NAD in which pods come up, then an SVI will map to the NAD.
Note that secondary subnets can be configured under the subnets section. L3Out instance specific configuration can be done under vrf/tenant/l3out.

Both SVI types – floating and conventional are supported. 
Note that Floating L3Out has an anchor node limit set to 6 and ACI nodes added after this, will be considered non-anchor nodes.
For conventional SVI, maximum nodes setting defaults to 10 and can be changed directly in the NetworkFabricL3Configuration CR.
VRF and Tenant are always required to be pre-existing.
Two workflows are possible as part of the L3 design for CNO. 

Workflow 1: corresponds to pre-created l3out by user. This means all the traditional l3out configuration that is not tied to actual l3out physical link, like route-maps, external epgs would be configured by customer. Actual l3out physical link and nodes under node profile would be configured by the `NetworkFabricL3Configuration` CR.

Workflow 2: corresponds to l3out created by CNO. Entire config that is traditionally under an ACI L3Out instance can be configured with the `NetworkFabricL3Configuration` CR.

### 10.1 Using pre-existing ACI L3Out

Following CR creates a floating svi on pre-existing l3out pre-l3out-1 in common ACI tenant and pre-existing vrf pre-vrf-1.
This is the preferred workflow as it is likely that user has an external peering connection on the same SVI and/or other custom config,
not all of which may be available in the `NetworkFabricL3Configuration` CR.
This is minimal config, more fields can be found looking at the `NetworkFabricL3Configuration` CRD.
```
apiVersion: aci.fabricattachment/v1
kind: NetworkFabricL3Configuration
metadata:
  name: networkfabricl3configuration
spec:
  vrfs:
  - directlyConnectedNetworks:
    - bgpPeerPolicy:
        enabled: true
        peerASN: 64515
      encap: 102
      l3OutName: pre-l3out-1
      l3OutOnCommonTenant: true
      primarySubnet: 192.168.100.0/24
      # Use type as svi for conventional svi
      sviType: floating_svi
      useExistingL3Out: true
    vrf:
      common-tenant: true
      name: pre-vrf-1

```
In response, CNO will show the related configuration that has been pushed to the APIC in the CR's status.
The status below shows that atleast one NAD used the vlan 102 and this caused the floating SVI to be created in
ACI. The nodes that were part of the static path associated with that NAD were leaf 101 and 102 and they were added to the l3out as L3Out router nodes.
Pods that come up on this NAD can peer with ACI leaf nodes 101 and 102 with the addresses  192.168.100.247 and 192.168.100.248 using ASN 64515.
More on which addresses, pods should use to peer, will follow in section 10.4.
```
...
status:
  vrfs:
  - directlyConnectedNetworks:
    - bgpPeerPolicy:
        enabled: true
        peerASN: 64515
        secret:
          name: ""
          namespace: ""
      encap: 102
      l3OutName: pre-l3out-1
      l3OutOnCommonTenant: true
      nodes:
      - nodeRef:
          nodeId: 101
          podId: 1
        primaryAddress: 192.168.100.247/24
      - nodeRef:
          nodeId: 102
          podId: 1
        primaryAddress: 192.168.100.248/24
      primarySubnet: 192.168.100.0/24
      subnets:
      - connectedSubnet: 192.168.100.0/24
        floatingAddress: 192.168.100.254/24
        secondaryAddress: 192.168.100.253/24
      sviType: floating_svi
      useExistingL3Out: true
    tenants:
    - commonTenant: true
      - name: pre-l3out-1
        podRef:
          podId: 1
        rtrNodes:
        - nodeRef:
            nodeId: 101
            podId: 1
          rtrId: 101.101.0.101
        - nodeRef:
            nodeId: 102
            podId: 1
          rtrId: 102.102.0.102
    vrf:
      common-tenant: true
      name: pre-vrf-1
```

### 10.2 Using generated ACI L3Out

Following CR creates a conventional svi on a new l3out auto-l3out-1 in common ACI tenant and pre-existing vrf pre-vrf-1.
This is minimal config, more fields can be found looking at the `NetworkFabricL3Configuration` CRD.
```
apiVersion: aci.fabricattachment/v1
kind: NetworkFabricL3Configuration
metadata:
  name: networkfabricl3configuration
spec:
  vrfs:
  - directlyConnectedNetworks:
     - bgpPeerPolicy:
        enabled: true
        peerASN: 64516
      encap: 103
      l3OutName: auto-l3out-1
      l3OutOnCommonTenant: true
      primarySubnet: 192.168.120.0/24
      sviType: svi
    tenants:
    - commonTenant: true
      l3OutInstances:
      - externalEpgs:
        - name: default
          policyPrefixes:
          # At least one policyprefix is needed for forwarding to work.
          # Should include the subnet that is required, unlikely to be 0.0.0.0/0
          - subnet: 0.0.0.0/0
        name: auto-l3out-1
        podRef:
          podId: 1
    vrf:
      common-tenant: true
      name: pre-vrf-1
```
In response, CNO will show the related configuration that has been pushed to the APIC in the CR's status.
The status below shows that atleast one NAD used the vlan 103 and this caused the conventional SVI to be created in
ACI. The nodes that were part of the static path associated with that NAD were leaf 101 and 102 and they were added to the l3out as L3Out router nodes along with the static path. Pods that come up on this NAD can peer with ACI leaf nodes 101 and 102 with the addresses 192.168.120.243 and 192.168.120.244 using ASN 64516.
More on which addresses, pods should use to peer, will follow in section 10.4.
```
...
status:
  vrfs:
    - directlyConnectedNetworks:
      - bgpPeerPolicy:
          enabled: true
          peerASN: 64516
          secret:
            name: ""
            namespace: ""
        encap: 103
        l3OutName: auto-l3out-1
        l3OutOnCommonTenant: true
        nodes:
        - nodeRef:
            nodeId: 102
            podId: 1
          primaryAddress: 192.168.120.244/24
        - nodeRef:
            nodeId: 101
            podId: 1
          primaryAddress: 192.168.120.243/24
        primarySubnet: 192.168.120.0/24
        subnets:
        - connectedSubnet: 192.168.120.0/24
          secondaryAddress: 192.168.120.253/24
        sviType: svi
      tenants:
      - commonTenant: true
        l3OutInstances:
        - externalEpgs:
          - contracts: {}
            name: default
            policyPrefixes:
            - subnet: 0.0.0.0/0
          name: auto-l3out-1
          podRef:
            podId: 1
          rtrNodes:
          - nodeRef:
              nodeId: 102
              podId: 1
            rtrId: 102.102.0.102
          - nodeRef:
              nodeId: 101
              podId: 1
            rtrId: 101.101.0.101
      vrf:
        common-tenant: true
        name: pre-vrf-1
```
### 10.3 Kubernetes node to ACI leaf peer mapping

In response to the `NetworkFabricL3Configuration` CR, CNO also publishes a status only cluster-scoped CR called `NodeFabricNetworkL3Peer`.
This CR shows the mapping of a k8s Node when used in the context of a NAD to the corresponding ACI L3Out Node. Considering only the config in section 10.1, the
resulting CR will look like below. This means that pods on  k8s nodes k8s-node1 and k8s-node2 can peer with fabric l3out nodes node-101 and node-102, while using encap 102 and the corresponding peerInfo under the fabric node.

```
apiVersion: aci.fabricattachment/v1
kind: NodeFabricNetworkL3Peer
metadata:
  name: nodefabricnetworkl3peer
status:
  nadRefs:
  - nad:
      name: macvlan-net2
      namespace: default
    nodes:
    - fabricL3Peers:
      - encap: 102
        fabricNodeIds:
        - 101
        - 102
        podId: 1
      nodeName: k8s-node1
    - fabricL3Peers:
      - encap: 102
        fabricNodeIds:
        - 101
        - 102
        podId: 1
      nodeName: k8s-node2
  peeringInfo:
  - asn: 64515
    encap: 102
    fabricNodes:
    - nodeRef:
        nodeId: 102
        podId: 1
      primaryAddress: 192.168.100.248/24
    - nodeRef:
        nodeId: 101
        podId: 1
      primaryAddress: 192.168.100.247/24
    secret:
      name: ""
      namespace: ""

```

### 10.4 Webhook based environment variable insertion for router pods

Fabric peering information that a specific pod can use, can be published as environment variables using the webhook in CNO. Webhook uses the 
`NodeFabricNetworkL3Peer` CR mentioned in section 10.3.
To enable this functionality, required pod should be annotated with `netop-cni.cisco.com/fabric-l3peer-inject: <list of comma-separated NAD names needing insertion>`.
In addition, the container that needs environment variables needs to be named with the suffix `fabric-peer`. This suffix can be configured through
acc-provision. Using the example in section 10.1, when the pod comes up using the specific NAD(macvlan-net2), the following environment variables will be injected into the relevant pod.
 
```
BGP_PEERING_ENDPOINTS_MACVLAN-NET2=192.168.100.247/24,192.168.100.248/24
BGP_ASN_MACVLAN-NET2=64515
BGP_SECRET_PATH_MACVLAN-NET2=
```

Note that in case of floating svi, in case the pod comes up on a k8s node that is not directly connected to any ACI leaf with a primary address configured ( k8s node connected to non-anchor node), then all the anchor node peers will be published in the list.  

### 10.5 Caveats

* BGP Route-maps are not configurable through the CR as the options are numerous and making entire set available in the CR, may not be practical.
Based on user feedback, most common options can be made available.
* In case of floating svi, an affinity can be set for a k8s node with a fabric anchor node, so only those addresses appear in the injected environment variables, instead of publishing all the anchor node addresses as is currently the case. This would be taken up in a future release.   
* While it is possible to use more than one vlan inside a NAD, the pod that uses this NAD needs to pick a vlan to use, in order for the webhook to apply the corresponding peering configuration. Also a given container would only need peering info for a specific NAD, if the pod is part of multiple additional networks.  Both of these can be part of another annotation on the pod. This may not be common usage, so at the moment the lowest encap under a NAD is picked and peering information from all the NADs in the inject list is added to the named container. Based on user feedback, CNO will consider the annotation support in a following release for these features.

