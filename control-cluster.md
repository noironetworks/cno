# CNO
Cisco Network Operator (CNO) - An Operator for managing networking for Kubernetes clusters and more (formerly known as Cisco Kubernetes Operator, CKO).

> This branch of CNO contains the latest features and fixes but is still under development. Previous stable release (0.9.0) can be found [here](https://github.com/noironetworks/cko/tree/stable/0.9.0). 

# Table of Contents

- [1. Introduction](#1-introduction)
- [2. Features](#2-features)
  - [2.1 Supported Integrations](#21-supported-integrations)
  - [2.2 Under Development](#22-under-development)
- [3. Deploying CNO](#3-deploying-cno)
  - [3.1 Control Cluster](#31-control-cluster)
    - [3.1.1 Prerequisites](#311-prequisites)
    - [3.1.2 Install cert-manager](#312-install-cert-manager)
    - [3.1.3 Create Secret for Github access](#313-create-secret-for-github-access)
    - [3.1.4 Deploy using Helm](#314-deploy-using-helm)
  - [3.2 Workload Cluster](#32-workload-cluster)
    - [3.2.1 Create Secret for Github access](#321-create-secret-for-github-access)
    - [3.2.2 Deploy Manifests](#322-deploy-manifests)
- [4. Using CNO](#4-using-cno)
  - [4.1 Workflows](#41-workflows)
    - [4.1.1 Fabric Onboarding](#411-fabric-onboarding)
      - [4.1.1.1 Fabric Identity](#4111-fabric-identity)
      - [4.1.1.2 Fabric Resources for Brownfield Clusters](#4112-fabric-resources-for-brownfield-clusters)
      - [4.1.1.3 Fabric Resources for Greenfield Clusters](#4113-fabric-resources-for-greenfield-clusters)
        - [4.1.1.3.1 Subnets and VLANs](#41131-subnets-and-vlans)
        - [4.1.1.3.2 External Connectivity](#41132-external-connectivity)
        - [4.1.1.3.3 Topology](#41133-topology)
        - [4.1.1.3.4 BGP](#41134-bgp)
      - [4.1.1.4 Allocation Status](#4114-allocation-status)
    - [4.1.2 Brownfield Clusters](#412-brownfield-clusters)
      - [4.1.2.1 Unmanaged CNI](#4121-unmanaged-cni)
      - [4.1.2.2 Managed CNI](#4122-managed-cni)
    - [4.1.3 Greenfield Clusters](#413-greenfield-clusters)
    - [4.1.4 Managing Clusters as a Group](#414-managing-clusters-as-a-group)
    - [4.1.5 Managing Clusters Individually](#415-managing-clusters-individually)
    - [4.1.6 Customizing Default Behaviors](#416-customizing-default-behaviors)
    - [4.1.7 Upgrade Managed CNI Operators](#417-upgrade-managed-cni-operators)
    - [4.1.8 Upgrade CNO in Workload Cluster](#418-upgrade-cno-in-workload-cluster)
    - [4.1.9 Deleting CNO from Workload Cluster](#419-deleting-cno-from-workload-cluster)
    - [4.1.10 Upgrade Control Cluster](#4110-upgrade-control-cluster)
  - [4.2 API Reference](#42-api-reference)
  - [4.3 Sample Configuration](#43-sample-configuration)
- [5. Observability & Diagnostics](#5-observability--diagnostics)
  - [5.1 Diagnostics on the Control Cluster](#51-diagnostics-on-the-control-cluster)
  - [5.2 Diagnostics on the Workload Clusters](#52-diagnostics-on-the-workload-cluster)
- [6. Troubleshooting](#6-troubleshooting)
  - [6.1 Brownfield case](#61-brownfield-case)
- [7. Contributing](#7-contributing)
  - [7.1 Repositories](#71-repositories)
  - [7.2 Contributing to CNO](#72-contributing-to-cno)
  - [7.3 Experimenting with Control Cluster](#73-experimenting-with-control-cluster)
- [Appendix](#Appendix)
  - [Single Node Control Cluster](#single-node-control-cluster)
  - [Control Cluster Install Configuration](#control-cluster-install-configuration)
  - [CNO Cleanup in Workload Cluster](#cno-cleanup-in-workload-cluster)

## 1. Introduction

The [CNCF Cloud Native Landscape](https://landscape.cncf.io/?grouping=category) illustrates the rich and rapidly evolving set of projects and components in the Cloud Native Networking domain. Using these components requires installation and operational knowledge of each one of those. It also leaves the burden on the user to harmonize the configuration across the networking layers and components to ensure that everything works in sync. This gets even more complicated when you consider that most production solutions run applications which are deployed across multiple Kubernetes clusters.

CNO aims to alleviate this complexity and reduce the operational overhead by:
* Automation - Providing resource management across network resources and automating the composition of networks and services
* Observability - Providing observability by correlating between clusters and infrastructure, by centralized data collection and eventing, and by health check and reporting at global level
* Operations - Providing operational benefits via centralized network governance, migration of workloads, and cost optimization across cluster sprawl
* Security - Providing multi-cluster security by federating identity across domains

CNO achieves this by defining simple abstractions to meet the needs of he following persona:
* Kubernetes Admin - responsible for managing the cluster
* Cloud Admin - responsible for the coordinating the infrastructure needs of a cluster
* Network Admin - responsible for the network infrastructure

These abstractions are modeled to capture the user's intent and then consistently apply it across the infrastructure components. The abstractions are:
* ClusterProfle - defined by the Kubernetes Admin to express the network needs for the cluster they intend to manage
* ClusterGroupProfile - defined by the Cloud Admin to match the networking needs of all matching clusters with network and other infrastructure
* ClusterNetworkProfile - defined by the Cloud Admin to match the specific needs of one cluster
* FabricInfra - defined by the Network Admin to model each discrete physical or virtual network infrastructure unit that provides pod, node and external networking capabilities to the cluster

The diagram below illustrates the relationship between these abstactions.

![CNO Resources](docs/user-guide/diagrams/class-diagram.drawio.png)

The abstractions ensure that these persona can seamlessly collaborate to dynamically satisfy the networking needs of the set of clusters they manage. The abstractions are flexible and can be applied to a group of clusters which can be managed as a whole, or can be used to create individual snowflakes. 

The diagram below illustrates a typical CNO deployment comprising of one Control Cluster and one or more Workload Clusters with the following CNO components:
* A centralized "Org Operator" for identity and resource management 
* One or more "Fabric Operators" for network infrastructure automation and kubernetes manifest generation
* "Per Cluster Operators" for managing the lifecycle of network components in the Workload Cluster

The Workload Cluster runs the user's applications. The lifecycle of all the clusters is managed by the user; CNO only requires that specific operators be run in these. The lifecycle of CNO in the Workload Cluster, once deployed, is managed by the Control Cluster.

![Control and Workload Cluster](docs/user-guide/diagrams/control-and-workload-clusters.drawio.png)

## 2. Features
This release includes the following features:

* An Extensible Framework for CNI Lifecycle Management and Reporting
* Centralized Management comprising of
    * Network Data Model
    * CNI Asset Generation
    * CNI Upgrades
    * Connectivity Reporting

### 2.1 Supported Integrations
This release has support for:

* ACI Fabric 5.2
* ACI-CNI 5.2.3.5
* Calico CNI with Tigera Operator 3.23
* OCP 4.10
* Kubernetes 1.22, 1.23, 1.24

### 2.2 Roadmap Items Under Development
Support for the following technologies and products is being actively pursued:

| Feature       |  Product                                                 |
|---------------|----------------------------------------------------------|
| Network Infra | * Cisco Nexus Standalone (NDFC)<br> * AWS EKS            |
| CNI           | * AWS VPC<br> * Cilium<br> * OpenShift SDN               |
| Distributions | * Rancher                                                |
| Service Mesh  | * Cisco Calisti<br> * Istio<br> * OpenShift Service Mesh |
| Loadbalancer  | * MetalLB                                                |
| Ingress       | * NGINX                                                  |
| Monitoring    | * Prometheus<br> * Open Telemetry                        |
| DPU           | * Nvidia Bluefield                                       |

The above list is not comprehensive and we are constantly evaluating and adding new features for support based on user demand.

## 3. Deploying CNO
CNO requires one Control Cluster to be deployed with the CNO operators before any Workload Clusters can be managed by CNO. Existing Workload Clusters with a functioning CNI can be imported into CNO.

### 3.1 Control Cluster
This section describes the setup of the Control Cluster that manages the Workload Clusters.

A new Control Cluster can be deployed using [this script](scripts/install-control-cluster.sh):

```bash
./install-control-cluster.sh \
--repo <git-repo-URL> \
--branch <git-branch> \
--dir <top-level-dir-name> \
--github_pat <PAT> \
--git_user <git-username> \
--git_email <git-user-email> \
--http_proxy <http://example.proxy.com:port> \
--https_proxy <http://example.proxy.com:port> \
--no_proxy <localhost,127.0.0.1>
```

The Git repo is used by CNO to store configuration and status information. The http_proxy, https_proxy and no_proxy are optional arguments and need to be supplied if the host on which you are deploying the Control Cluster requires a proxy to connect to the Internet.

The following subsections describe the individual steps to install the Control Cluster if customizations are desired in the Control Cluster installation. If the Control Cluster is already deployed using the above script, please skip the following subsections and proceed to the [Workload Cluster](#32-workload-cluster).

#### 3.1.1 Prerequisites
* A functional Kubernetes cluster with reachability to the ACI Fabric that will serve as the Control Cluster. A single node cluster is adequate, refer to [Appendix](#single-node-control-cluster) for a quick guide on how to set up one.
* kubectl
* Helm

#### 3.1.2 Install cert-manager

``` bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.10.0 \
  --set installCRDs=true \
  --wait
```

#### 3.1.3 Create Secret for Github access
CNO follows the [GitOps](https://www.weave.works/technologies/gitops/) model using [Argo CD](https://github.com/argoproj/argo-cd) for syncing configuration between Control and Workload clusters. The Git repository details can be provided as shown below. The configuration below assumes that the Git repository is hosted in Github. You can optionally add the HTTP_PROXY details if your clusters require it to communicate with Github.

```bash
kubectl create ns netop-manager

kubectl create secret generic cko-config -n netop-manager \
--from-literal=repo=https://github.com/<ORG>/<REPO> \
--from-literal=dir=<DIR> \
--from-literal=branch=<BRANCH NAME> \
--from-literal=token=<GITHUB PAT> \
--from-literal=user=<GIT USER> \
--from-literal=email=<GIT EMAIL> \
--from-literal=http_proxy=<HTTP_PROXY> \
--from-literal=https_proxy=<HTTPS_PROXY> \
--from-literal=no_proxy=<add-any-other-IP-as-needed>,10.96.0.1,.netop-manager.svc,.svc,.cluster.local,localhost,127.0.0.1,10.96.0.0/16,10.244.0.0/16,control-cluster-control-plane,.svc,.svc.cluster,.svc.cluster.local

kubectl create secret generic cko-argo -n netop-manager \
--from-literal=url=https://github.com/<ORG>/<REPO> \
--from-literal=type=git  \
--from-literal=password=<GIT PAT> \
--from-literal=username=<GIT USER> \
--from-literal=proxy=<HTTP_PROXY>

kubectl label secret cko-argo -n netop-manager 'argocd.argoproj.io/secret-type'=repository
```

#### 3.1.4 Deploy using Helm
Use the template provided in the [Appendix](#control-cluster-install-configuration) to create the ```my_values.yaml``` used during the helm install of CNO in the control cluster. It sets the relevant image registries, tags and optionally HTTP-Proxy configuration.

``` bash

helm repo add cko https://noironetworks.github.io/netop-helm
helm repo update
helm install netop-org-manager cko/netop-org-manager -n netop-manager --create-namespace --version 0.9.1 -f my_values.yaml --wait
```

Argo CD is automatically deployed in the Control Cluster (in the netop-manager namespace) and in the Workload Cluster (in the netop-manager-system namespace). By default Argo CD starts reconciliation with the git repository every [180s](https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/argocd-cm.yaml#L283). The reconciliation time depends on the number of objects being synchronized and can potentially take longer at times. If more frequent synchronization is required you can follow the process described [here](https://www.buchatech.com/2022/08/how-to-set-the-application-reconciliation-timeout-in-argo-cd/) to reduce time interval between the reconciliation start times.

### 3.2 Workload Cluster
CNO handles the integration and management of the CNI in the Workload Clusters as two distinct cases:
* Brownfield - The Workload Cluster already exists with a fully functioning and supported CNI. An import workflow is used to make the CNI in such a cluster visible to the Control Cluster. In the Brownfield case CNO further provides the choice of (a) only observing, or (b) fully managing the CNI, and referred to as "Unmanaged" or "Managed" states respectively. The import workflow upon successful completion will always result in the "Unmanaged" state. The CNI can be moved to a "Managed" state through an explicit action by the user. Please refer to the [Brownfield case workflow](#412-brownfield-clusters) section for proceeding further.
* Greenfield - A new Workload Cluster needs to be installed. The network configuration for this planned cluster is initiated from the Control Cluster by creating the relevant ClusterProfile and/or ClusterGroupProfile/ClusterNetworkProfile CRs. Please refer to the [Greenfield case workflow](#413-greenfield-clusters) section for proceeding further.

The rest of this section provides the steps for deploying the CNO components in the Workload Cluster. However, prior to any action in the Workload Cluster, the user needs to identify and initiate the appropriate [Brownfield case workflow](#412-brownfield-clusters) or the [Greenfield case workflow](#413-greenfield-clusters) at least once for the Workload Cluster under consideration. 

#### 3.2.1 Create Secret for Github access
Provide the same Git repository details as those in the Control Cluster.

```bash
kubectl create ns netop-manager-system

kubectl create secret generic cko-config -n netop-manager-system \
--from-literal=repo=https://github.com/<ORG>/<REPO> \
--from-literal=dir=<DIR> \
--from-literal=branch=<BRANCH NAME> \
--from-literal=token=<GITHUB PAT> \
--from-literal=user=<GIT USER> \
--from-literal=email=<GIT EMAIL> \
--from-literal=systemid=<SYSTEM ID> \
--from-literal=http_proxy=<HTTP_PROXY> \
--from-literal=https_proxy=<HTTPS_PROXY> \
--from-literal=no_proxy=<NO_PROXY>

kubectl create secret generic cko-argo -n netop-manager-system \
--from-literal=url=https://github.com/<ORG>/<REPO> \
--from-literal=type=git  \
--from-literal=password=<GIT PAT> \
--from-literal=username=<GIT USER> \
--from-literal=proxy=<HTTP_PROXY>

kubectl label secret cko-argo -n netop-manager-system 'argocd.argoproj.io/secret-type'=repository
```

* When importing a cluster with a functional ACI CNI (Brownfield case), the ```systemid``` parameter in the above secret should match the one mentioned in the ```system_id``` section of the acc-provision input file. In all other cases, the ```systemid``` can be set to a relevant name that establishes an identity for this cluster (hyphen is not permitted in the name per ACI convention).

* If the HTTP proxy is set, the following need to be added to the no-proxy configuration:

    * In the Kubernetes case: 
    ```
    10.96.0.1,localhost,127.0.0.1,172.30.0.1,172.30.0.10,<node-IPs>,<node-host-names>,<any-other-IPs-which-need-to-be-added>
    ```

    * In the OpenShift case: 
    ```
    oauth-openshift.apps.<based-domain-from-install-config-yaml>.local,console-openshift-console.apps.<based-domain-from-install-config-yaml>.local,downloads-openshift-console.apps.<based-domain-from-install-config-yaml>.local,localhost,127.0.0.1,172.30.0.1,172.30.0.10,<node-IPs>,<node-host-names>,<any-other-IPs-which-need-to-be-added>
    ```

    * In the Calico CNI case:\
    In addition to the above, add ```.calico-apiserver.svc```

#### 3.2.2 Deploy Manifests

For OpenShift Cluster:

``` bash

kubectl apply -f https://raw.githubusercontent.com/noironetworks/netop-manifests/cko-mvp-1/workload/netop-manager-openshift.yaml
kubectl create -f https://raw.githubusercontent.com/noironetworks/netop-manifests/cko-mvp-1/workload/platformInstaller.yaml
```

For non-OpenShift Cluster:

``` bash

kubectl apply -f https://raw.githubusercontent.com/noironetworks/netop-manifests/cko-mvp-1/workload/netop-manager.yaml
kubectl create -f https://raw.githubusercontent.com/noironetworks/netop-manifests/cko-mvp-1/workload/platformInstaller.yaml
```

## 4. Using CNO

### 4.1 Workflows

#### 4.1.1 Fabric Onboarding
Each network infrastructure unit (referred to as a fabric) that is managed as an independent entity, is modeled in CNO using the FabricInfra CRD. The network admin creates a FabricInfra CR to establish the identity of the of that fabric, and allows the network admin to specify the set of resources available to be consumed on that fabric. CNO reserves these resources in the FabricInfra on the cluster's behalf, and performs the necessary provisioning on the fabric to enable the networking for the cluster.

##### 4.1.1.1 Fabric Identity
The following fields are required when creating the FabricInfra:

```bash
  credentials:
    hosts:
    - <APIC-IP-1>
    - ...
    secretRef:
      name: apic-credentials
  fabric_type: aci
  infra_vlan: <infra-vlan-value>
```

The secretRef property refers to a Kubernetes Secret for the APIC username and password. It can be created as follows (note that the name of the Secret should match that in the secretRef, in this case the name is ```apic-credentials```):

```bash
kubectl create secret -n netop-manager generic apic-credentials --from-literal=username=<APIC_USERNAME> --from-literal=password=<APIC_PASSWORD>
```

The username provided above should have privileges to access at least the "common" tenant on the APIC.

##### 4.1.1.2 Fabric Resources for Brownfield Clusters
Fabric requirements for subnets, VLAN and external connectivity do not need to be explicitly defined in the FabricInfra spec for imported clusters. Since these resources are already in use on the fabric, CNO will learn them at the time of importing a particular cluster, and automatically reserve them in the FabricInfra. These resources will however not be released to the available pool when the imported cluster is removed from CNO.

If dealing only with brownfield clusters no further configuration is required to be specified in the FabricInfra.

##### 4.1.1.3 Fabric Resources for Greenfield Clusters
Fabric resources needed for a cluster can be individually specified as a part of the ClusterProfile or the ClusterNetworkProfile. Alternatively, fabric resources needed for multiple clusters can be optionally specified as pools in the FabricInfra spec such that allocations can be made to individual clusters on demand. If the fabric resources are not specified as a part of the ClusterProfile or ClusterNetworkProfile, the required resources are allocated from the available pools of resources.

##### 4.1.1.3.1 Subnets and VLANs
The snippet below shows how subnets used for node networks, multicast (for ACI-CNI), external connectivity, and VLANs (for ACI-CNI) for node and service networks can be specified: 

```bash
  ...
  mcast_subnets:
    - 225.114.0.0/16
    - 225.115.0.0/16
  internal_subnets:
    - 1.100.101.1/16
    - 2.100.101.1/16
    - 10.5.0.1/16
    - 10.6.0.1/16
    - 20.2.0.1/16
    - 20.5.0.1/16
  external_subnets:
    - 10.3.0.1/16
    - 10.4.0.1/16
    - 20.3.0.1/16
    - 20.4.0.1/16
  vlans:
    - 101
    - 102
    - 103
    - 104
    ...
```

If fabric resources are being allocated from the FabricInfra pools, at least two internal subnets, one multicast subnet, two external subnets, and two VLANs need to be available for a greenfield cluster with ACI-CNI, or at least one external subnet needs to be available for a greenfield cluster with Calico CNI.

CNO will automatically pick available resources from resource pools. Resources are picked in round-robin fashion. CNO currently does not partition the subnets, so each subnet specified in the resource pool is treated as an indivisible pool. If smaller subnets are desired, they should be listed as individual subnets.

Note: The pod subnet is not picked from the pool since its local to a cluster. It defaults to ```10.2.0.0/16``` and can be overridden in the ClusterProfile, or ClusterGroupProfile, or ClusterNetworkProfile.

##### 4.1.1.3.2 External Connectivity
The kubernetes_node-to-fabric and fabric-to-external connectivity on each fabric is encapsulated in the context property of the FabricInfra. For example in the case of the ACI fabric, the following example shows the context and its constituent properties:

```bash
  ...
  contexts:
    context-1:
      aep: bm-srvrs-aep
      l3out:
        name: l3out-1
        external_networks:
        - l3out-1_epg
      vrf:
        name: l3out-1_vrf
        tenant: common
    ...
```

At least one context is required for each cluster, and all fields in the context are required. CNO currently does not create ACI AEP, VRF, and L3out; they have to be created by the network admin and referenced in the above configuration.

##### 4.1.1.3.3 Topology
The kubernetes_nodes and top-of-the-rack interconnection topology per fabric is captured in the topology section as shown below:

```bash
  topology:
    rack:
    - id: 1
      aci_pod_id: 1
      leaf:
      - id: 101
      - id: 102
      node:
      - name: k8s-node1
      - name: k8s-node2
    - id: 2
      aci_pod_id: 1
      leaf:
      - id: 103
      - id: 104
      node:
      - name: k8s-node3
      - name: k8s-node4
```

The specification of the topology section is required when the fabric is intended to be used for clusters with the Calico CNI, its optional if only the ACI-CNI is intended to be deployed. The rack and aci_pod_id properties are user defined, where as the leaf IDs point to their corresponding values on the fabric.

Note: If deploying more than one cluster with Calico CNI, current limitations require the topology to be specified as a part of the ClusterNetworkProfile.

##### 4.1.1.3.4 BGP
The BGP configuration for the fabric can be specified as follows:

```bash
  bgp:
    remote_as_numbers:
      - 64512
    aci_as_number: 2
```

The specification of the bgp section is required when the fabric is intended to be used for the Calico CNI, its optional if only the ACI-CNI is intended to be deployed.

##### 4.1.1.4 Allocation Status
All available and allocated resource are ref![image](https://github.com/user-attachments/assets/aa0d3947-2ac0-44b1-b178-7f131cd1927c)
![image](https://github.com/user-attachments/assets/77dab8c1-0c99-462d-94ed-071e5a888f04)
![image](https://github.com/user-attachments/assets/1f45da46-14ff-4145-8a2d-becbaebc674a)
lected in the status field of the FabricInfra as shown in the example below:

```bash
status:
  allocated:
  - cluster_profile: bm2acicni
    context:
      aep: bm-srvrs-aep
      l3out:
        external_networks:
        - sauto_l3out-1_epg
        name: sauto_l3out-1
      vrf:
        name: sauto_l3out-1_vrf
        tenant: common
    external_subnets:
    - 10.3.0.1/16
    - 10.4.0.1/16
    internal_subnets:
    - 1.100.101.1/24
    - 10.6.0.1/16
    - 10.5.0.1/16
    mcast_subnets:
    - 225.114.0.0/16
    vlans:
    - 101
    - 102
  available:
    contexts:
      context-2:
        aep: bm-srvrs-calico-aep
        l3out:
          external_networks:
          - sauto_l3out-3_epg
          name: sauto_l3out-3
        vrf:
          name: sauto_l3out-3_vrf
          tenant: common
    external_subnets:
    - 20.3.0.1/16
    - 20.4.0.1/16
    internal_subnets:
    - 1.100.101.1/16
    - 2.100.101.1/16
    - 20.2.0.1/16
    - 20.5.0.1/16
    mcast_subnets:
    - 225.115.0.0/16
    remote_as_numbers:
    - 64512
    vlans:
    - 103
    - 104
```

The complete API spec for the FabricInfra can be found here: [CRD](docs/control-cluster/api_docs.md#fabricinfra)

An example of the FabricInfra CR can be found here: [Example CR](config/samples/aci-cni/kubernetes/fabricinfra.yaml)

#### 4.1.2 Brownfield Clusters
Existing clusters with a functional CNI provisioned with acc-provision flavors for release 5.2.3.4 can be imported into CNO. Please refer to the following documentation for installing a cluster with ACI CNI or Calico CNI on ACI:
* ACI CNI
    * [Cisco ACI and Kubernetes Integration](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/kb/b_Kubernetes_Integration_with_ACI.html#task_ggz_svz_r1b)
    * [Installing OpenShift 4.10 on Bare Metal](https://www.cisco.com/c/en/us/td/docs/dcn/aci/containers/installation/installing-openshift-4-10-on-baremetal.html)
    * [Installing OpenShift 4.10 on OpenStack 16.2](https://www.cisco.com/c/en/us/td/docs/dcn/aci/containers/installation/openshift-on-openstack/installing-openshift-4-10-on-openstack-16-2.html)
    * [Installing OpenShift 4.10 on VMware vSphere](https://www.cisco.com/c/en/us/td/docs/dcn/aci/containers/installation/openshift-on-vsphere/installing-openshift-4-10-on-vmware-vsphere.html)
* Calico CNI
    * [Cisco ACI and Calico 3.23.2 Integration](https://www.cisco.com/c/en/us/td/docs/dcn/aci/containers/installation/cisco-aci-calico-integration/cisco-aci-with-calico-integration.html)

 The imported cluster will initially have its CNI in an observed, but unmanaged, state by CNO. After succesfully importing the cluster, the CNI can be transitioned to a managed state after which the CNI's configuration and lifecycle can be completely controlled from the Control Cluster.

##### 4.1.2.1 Unmanaged CNI
* Pre-requisite: The network admin has on-boarded the fabric by creating a [FabricInfra CR](#4111-fabric-identity).

This workf![image](https://github.com/user-attachments/assets/396494fa-f0f6-49b8-ac7e-e712af42de24)
![image](https://github.com/user-attachments/assets/ba02a0db-a07b-410b-a6d1-5d0059906646)
![image](https://github.com/user-attachments/assets/696cabb4-7e3e-4f5c-94f8-c5423b7f59b2)
![image](https://github.com/user-attachments/assets/01c863c4-9789-4e21-bec1-efcd890d2062)
low is initiated in the Workload Cluster which needs to be imported.

The first step is to create the secrets to access a Github repo as shown [here](#321-create-secret-for-github-access).

Then apply the CNO workload cluster operator manifests as show [here](#322-deploy-manifests).

Once applied, the notification to import the cluster will be sent to the Control Cluster via Gitops. Once Argo CD syncs on the Control Cluster you will see the following following two resources getting created:

- [ClusterProfile](config/samples/aci-cni/kubernetes/imported/auto-clusterprofile.yaml) with name: ```auto-<cluster-name>```
- [ClusterNetworkProfile](config/samples/aci-cni/kubernetes/imported/auto-clusternetworkprofile.yaml) with name: ```auto-<cluster-name>```

The status of the imported cluster can now be tracked in the Control Cluster. 

##### 4.1.2.2 Managed CNI
* Pre-requisite: A cluster with an associated ClusterProfile is present (this will have happened if the import workflow was successful as described in the [previous section](#4121-unmanaged-cni)). 

Change the following in the relevant ClusterProfile:

```bash
...
  operator_config:
    mode: unmanaged
...
```

to:


```bash
...
  operator_config:
    mode: managed
...
```

This will trigger the workflow on the workload cluster once Argo CD syncs, such that the installed CNI will be managed by the Control Cluster.

#### 4.1.3 Greenfield Clusters
* Pre-requisite: The network admin has on-boarded the fabric by creating a [FabricInfra CR](#4111-fabric-identity). In addition, depending on the CNI that is intended to be deployed, additional FabricInfra configuration may be required as indicated in the section [Fabric Resources for Greenfield Clusters](#4113-fabric-resources-for-greenfield-clusters). As such, for existing ACI users already familiar with configuring the acc-provision input file, the [Brownfield workflow](#412-brownfield-clusters) might be preferable to on-ramp their clusters (even new clusters) as opposed to this Greenfield workflow.

* Note that in the Calico CNI case the topology model in the FabricInfra is currently restricted to specifying host-level connectivity only for a single Workload Cluster. This can be worked around by explicitly specifying the topology per cluster in the ClusterProfile or ClusterNetworkProfile CRs.

The user creates a simple ClusterProfile and specifies the CNI.

For ACI-CNI:
```bash
apiVersion: netop.mgr/v1alpha1
kind: ClusterProfile
metadata:
  name: <cluster-name>
  namespace: netop-manager
spec:
  cni: aci
```

For Calico-CNI:
```bash
apiVersion: netop.mgr/v1alpha1
kind: ClusterProfile
metadata:
  name: <cluster-name>
  namespace: netop-manager
spec:
  cni: calico
```

This will result in a choice of an available FabricInfra and the allocation of relevant resource on that fabric. On success, the ClusterProfile status will reflect the allocated resources such that it can be used in the cluster installation. For example:

```bash
...
status:
  aci_cni_config:
    version: 5.2.3.4
  cluster_network_profile_name: bm2acicni
  context:
    aep: bm-srvrs-aep
    l3out:
      external_networks:
      - sauto_l3out-1_epg
      name: sauto_l3out-1
    vrf:
      name: sauto_l3out-1_vrf
      tenant: common
  external_subnets:
  - 10.3.0.1/16
  - 10.4.0.1/16
  fabricinfra:
    context: context-1
    name: k8s-bm-2
  internal_subnets:
  - 1.100.101.1/24
  - 10.6.0.1/16
  - 10.5.0.1/16
  mcast_subnets:
  - 225.114.0.0/16
  node_subnet: 1.100.101.1/24
  node_vlan: 101
  operator_config:
    mode: unmanaged
    version: 0.8.0
  pod_subnet: 10.6.0.1/16
  ready: true
  vlans:
  - 101
  - 102
  workload_cluster_manifest_locations:
  - 'argo: https://github.com/networkoperator/CNOgitrepo/tree/test/workload/argo/bm2acicni'
  - 'operator: https://github.com/networkoperator/ckogitrepo/tree/test/workload/config/bm2acicni' 
```

When the ClusterProfile is deleted, the allocated resources are returned to the avaliable pool.

The ClusterProfile API also allows to specify all the configurable details with regards to the CNI if the user wants to choose specific values for the fields. For resources that are chosen from the FrabricInfra, those resources will have to be available in the FabricInfra.

The complete API spec for the ClusterProfile can be found here: [CRD](docs/control-cluster/api_docs.md#clusterprofile)

An example of the ClusterProfile CR can be found here: [Example CR](config/samples/aci-cni/kubernetes/clusterGroupProfile/clusterprofile_k8s.yaml)

Once the ClusterProfile CR is created successfully, the focus shifts to the Workload Cluster to deploy CNO operator by following these [instructions](#32-workload-cluster). This will result in CNO running in the Workload Cluster and which will in turn deploy the CNI.

#### 4.1.4 Managing Clusters as a Group

Create ClusterGroupProfile with common properties like CNI, Distro etc, set labels.

After creating the ClusterProfile for a cluster, set ClusterGroupProfileSelector to match ClusterGroupProfile's labels.

Also updates to properties such as CNI management modes (managed versus unmanaged), CNO version, and CNI versions can be done in the ClusterGroupProfile instead of individual clusters.

The complete API spec for the ClusterGroupProfile can be found here: [CRD](docs/control-cluster/api_docs.md#clustergroupprofile)

An example of the ClusterGroupProfile CR can be found here: [Example CR](config/samples/aci-cni/kubernetes/clusterGroupProfile/clustergroupprofile_k8s.yaml)

#### 4.1.5 Managing Clusters Individually

Create ClusterNetworkProfile CR with all the specific desired properties for this cluster.

Create ClusterProfile for cluster, set ClusterNetworkProfileSelector to match ClusterNetworkProfile's labels.

An example of the ClusterNetworkProfile CR can be found here: [Example CR](config/samples/aci-cni/kubernetes/clusterNetworkProfile/clusternetworkprofile.yaml)

#### 4.1.6 Customizing Default Behaviors
* ConfigMap for ClusterProfle global default settings: defaults-cluster-profile.yaml
* ConfigMap for FabricInfra global default settings: defaults-global-fabricinfra.yaml

#### 4.1.7 Upgrade Managed CNI Operators
Update CNI version in ClusterProfile,

For ACI CNI:

```bash
...
           config_overrides:
                  aci_cni_config:
                      ...
                      target_version: <>
                        ...
```

For Calico CNI:

```bash
...
           config_overrides:
                  calico_cni_config:
                      ...
                      target_version: <>
                        ...
```

#### 4.1.8 Upgrade CNO in Workload Cluster
Update CNO version in ClusterProfile by changing the following:

```bash
...
  operator_config:
    ...
    target_version: 0.9.1
...
```

#### 4.1.9 Deleting CNO from Workload Cluster
To avoid accidentally breaking the Workload Cluster, deletion of CNO and/or the CNI is an explicit step. Please refer to the cleanup instructions in the [Appendix](#cno-cleanup-in-workload-cluster) to initiate this cleanup. The ClusterProfile should be deleted only after the cleanup has been performed on the Workload Cluster. The configuration on the Fabric for this cluster is cleaned up after the ClusterProfile is deleted.

#### 4.1.10 Upgrade Control Cluster
To upgrade from 0.9.0 release to 0.9.1 use the following helm upgrade command using the template provided in the [Appendix](#control-cluster-install-configuration) to create the ```my_values.yaml```:
```bash
  helm upgrade --install netop-org-manager cko/netop-org-manager -n netop-manager --create-namespace --version 0.9.1 -f my_values.yaml --wait
```
To upgrade the CRDs use the following command:
```bash
  kubectl apply -f https://raw.githubusercontent.com/noironetworks/netop-manifests/cko-mvp-1/control/netop-org-manager-crd.yaml     
```

As an alternative to the above helm command, in the dev environment, the following command can be used to upgrade or install using custom built images and other custom options.
```bash
	helm upgrade --install netop-org-manager cko/netop-org-manager -n netop-manager --create-namespace --version ${CHART_VERSION} \
	--set image.tag=${VERSION} --set image.repository=${IMAGE_TAG_BASE} \
	--set fabricManagerImage.repository=quay.io/ckodev/netop-fabric-manager \
	--set image.pullPolicy=IfNotPresent
```

### 4.2 API Reference
* [User API](docs/control-cluster/api_docs.md)
* [Editable Properties](docs/control-cluster/property_update_docs.md)

### 4.3 Sample Configurations
* [ACI-CNI](config/samples/aci-cni)
* [Calico](config/samples/calico)

## 5. Observability & Diagnostics

CNO has built-in diagnostic tools that provides insights in to the state of network configuration on both control and workload cluster sides. 

### 5.1 Diagnostics on the Control Cluster

#### 5.1.1 Workload cluster configuration state on Control Cluster:

After creating new FabricInfra Custom Resource for new Data Center Fabric, the new pod will be created with a name: *netop-fabric-manager-<FABRIC_NAME>-<RANDOM_NUMBER>* in the *netop-manager* namespace. 

Verify whether the pod has been created, using following command:

```
kubectl get pods -n netop-manager
```

After applying ClusterGroupProfile, ClusterProfile Custom Resources (CR) and optionally Config_Map for ClusterNetworkProfile, the workload cluster's network configuration is represented as a Custom Resource *clusterinfoes.netop.mgr* in the Control Cluster. 

*Note* this is namespaced resource, which will be created in the namespace you specified during creation of control cluster. Use following command to verify network configuration for workload cluster:

```
kubectl describe clusterinfoes.netop.mgr -n netop-manager <WORKLOAD_CLUSTER_NAME> 
```

In addition, ClusterProfile Custom Resource (CR) should be populated with the network details in the *.status* field. Artifacts are picked up from the FabricInfra or ClusterNetworkProfile (Config_Map). In addition, the *.status.deployment_script* field of the FabricInfra CR, consist of the URL links to the specific files pushed to the Git repository. Those files will be later used to deploy network configuration to the workload cluster.

#### 5.1.2 Verify network resources used from the common pools:

FabricInfra Custom Resource defines pools of resources such as subnets or VLANs available to use by Workload Clusters. Verify allocated resources by checking FabricInfra Custom Resource status field:

```
kubectl describe fabricinfras.netop.mgr -n netop-manager <FABRIC_INFRA_NAME>
```

#### 5.1.3 Verification of connectivity to Git

Configuration of the workload cluster is packaged in to following files and pushed to the Git repository:
The folder structure is the following:

```
`-- workload
    |-- argo
    |   |-- <workload_cluster_name>
    |   |   `-- argo_app.yaml
    `-- config
        `-- <workload_cluster_name>
            |-- installer-networking.yaml
            |-- installer-platform.yaml
            |-- installer.yaml
            `-- netop-manager-deployment.yaml
```

After successfully creation of the workload cluster network configuration in the Control Cluster, those files will be pushed to the Git repository. New folder with the name of the workload cluster will be created.


### 5.2 Diagnostics on the Workload Cluster


#### 5.2.1 Verify CNI installation

Regardless of the CNI installed, verify that all pods get IP address and are in the Running or Completed state.

**Note**, that some pods running in the host network namespace and share node IP addresses - those will be running even without CNI installed. 


#### 5.2.2 View summary of CNI network configuration of Worlkoad Cluster

The CniOps Custom Resource tracks usage of the network resources, IP pools allocations to nodes. It also tracks and checks for any inconsistencies and stale objects. 

Use following command to verify status of the CNI:

```
kubectl describe cniops <CNIOPS_NAME>
```

This resource is not namespaced. Use *get* to list available resources.
The output consists of some base64 encrypted configuration, however check the *.Status* field for allocated IP's per node, or inconsistencies reported.

Example output:

```
Status:
  Cni Status:
    Ipam - Scan:  Checking IPAM for inconsistencies...

Loading all IPAM blocks...
Found 3 IPAM blocks.
 IPAM block 10.4.189.0/26 affinity=host:calico-gf-master:
 IPAM block 10.4.190.192/26 affinity=host:calico-gf-worker1:
 IPAM block 10.4.251.0/26 affinity=host:calico-gf-worker2:
IPAM blocks record 13 allocations.

Loading all IPAM pools...
  10.4.0.0/16
Found 1 active IP pools.

Loading all nodes.
Found 0 node tunnel IPs.

Loading all workload endpoints.
Found 13 workload IPs.
Workloads and nodes are using 13 IPs.

Looking for top (up to 20) nodes by allocations...
  calico-gf-worker1 has 9 allocations
  calico-gf-worker2 has 2 allocations
  calico-gf-master has 2 allocations
Node with most allocations has 9; median is 2

Scanning for IPs that are allocated but not actually in use...
Found 0 IPs that are allocated in IPAM but not actually in use.
Scanning for IPs that are in use by a workload or node but not allocated in IPAM...
Found 0 in-use IPs that are not in active IP pools.
Found 0 in-use IPs that are in active IP pools but have no corresponding IPAM allocation.

Check complete; found 0 problems.

    Ipam - Status:
+----------+-------------+-----------+------------+--------------+
| GROUPING |    CIDR     | IPS TOTAL | IPS IN USE |   IPS FREE   |
+----------+-------------+-----------+------------+--------------+
| IP Pool  | 10.4.0.0/16 |     65536 | 13 (0%)    | 65523 (100%) |
+----------+-------------+-----------+------------+--------------+

    Version:  Client Version:    v3.23.2
Git commit:        a52cb86db
Cluster Version:   v3.23.2
Cluster Type:      typha,kdd,k8s,operator,bgp,kubeadm

  Cni Type:     calico
  Cni Version:  3.23
  Ipam:
+----------+-------------+-----------+------------+--------------+
| GROUPING |    CIDR     | IPS TOTAL | IPS IN USE |   IPS FREE   |
+----------+-------------+-----------+------------+--------------+
| IP Pool  | 10.4.0.0/16 |     65536 | 13 (0%)    | 65523 (100%) |
+----------+-------------+-----------+------------+--------------+

  Managed State:        New
  Observed Generation:  1
  State:                Running
  Upgrade Status:
    Cni Upgrade State:  None
Events:                 <none>
```

#### 5.2.3 Connectivity Checker and Error Pods Reporting

*Connectivity Checker* is a tool that continuously verifies and reports various connectivity paths like:
- node to External
- node to Loadbalancer VIP
- node to NodePort
- node to ClusterIP
- node to node
- node to pod
- pod to External
- pod to Loadbalancer VIP
- pod to NodePort
- pod to ClusterIP
- pod to node
- pod to pod
- pod to service

Verify connectivity using following command:

```
kubectl -n nettools get conncheck -oyaml
```

*Error Pods Reporting* is another tool that reports Pods in the failed state:

```
kubectl -n nettools get epr -oyaml
```

It collects outputs from various fields like events and logs and displays in single place. 

## 6. Troubleshooting

### 6.1 Brownfield case
Following notifications are required from workload cluster to initiate cluster profile creation. Note that brownfield is only supported where the corresponding CNI is supported through acc-provision.

#### 6.1.1 CNI neutral notifications required for all CNI

* Observedops:
<cluster-name>-netop-manager-system-discovery-agent-network-function-netop-manager-io-v1alpha1: Type should be corresponding CNI (eg. cko-cni-aci, cko-cni-calico) for a brownfield clusterprofile of corresponding CNI.
 
* Canary Installer:
<cluster-name>-netop-manager-system-canary-installer-controller-netop-manager-io-v1alpha1: There is no origin resource for this notification and it will be active only as long as there is no installer with CNI spec available on the workload cluster. This is required for a brownfield auto cluster profile creation.

* Configmaps: 
<cluster-name>-aci-containers-system-acc-provision-config-configmap-core-v1: All of the fields here are important.

#### 6.1.2 ACI-CNI
 
* Configmaps:
<cluster-name>-aci-containers-system-aci-operator-config-configmap-core-v1: Flavor field is important here to reconstruct acc-provision input.

* Secrets:
<cluster-name>-aci-containers-system-aci-user-cert-secret-core-v1: The authorized user secret created while originally provisioning workload cluster is synced to avoid disruption.  

#### 6.1.3 Calico
No additional requirements.
 
Once corresponding notifications are available, the ClusterProfile and ClusterNetworkProfile will be automatically created:

```auto-<clustername>``` - [ClusterProfile](config/samples/aci-cni/kubernetes/imported/auto-clusterprofile.yaml)

```auto-<clustername>``` - [ClusterNetworkProfile](config/samples/aci-cni/kubernetes/imported/auto-clusternetworkprofile.yaml)


## 7. Contributing

### 7.1 Repositories
* Org Operator: [netop-org-manager](https://github.com/noironetworks/netop-org-manager)

* Fabric Operator: [netop-fabric-manager](https://github.com/noironetworks/netop-fabric-manager)

* Workload Cluster Operator: [netop-manager](https://github.com/noironetworks/netop-manager)

* Notification Types: [netop-types](https://github.com/noironetworks/netop-types)

* Manifest Generation: [acc-provision](https://github.com/noironetworks/acc-provision/tree/cko-mvp-1)

* Connectivity Checker: [nettools](https://github.com/noironetworks/nettools)

* System Tests: [acc-pytests](https://github.com/noironetworks/acc-pytests/tree/cko-mvp-1)

* Control Cluster Helm Chart: [netop-helm](https://github.com/noironetworks/netop-helm/releases)

* Workload Cluster Manifests: [netop-manifests](https://github.com/noironetworks/netop-manifests)

* User Documentation: [cno](https://github.com/noironetworks/cno)

### 7.2 Contributing to CNO

[Developer Guide](docs/dev-guide/dev-and-contribute.md)

### 7.3 Experimenting with Control Cluster
CNO Control Cluster can be deployed in a disconnected mode from all fabrics. Edit defaults-global-fabricinfra ConfigMap:

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: defaults-global-fabricinfra
  namespace: netop-manager
data:
  provision_fabric: "false"
```

This can also be set per fabric in the fabricinfra CR as "spec.provision_fabric: false":

```bash
apiVersion: netop.mgr/v1alpha1
kind: FabricInfra
metadata:
  name: k8s-bm-2
  namespace: netop-manager
  labels:
    fabric: on-prem-dev-infra
    site: bldg-15-lab
spec:
  ...
  provision_fabric: "false"
  ...
```

## Appendix

### Single Node Control Cluster
A simple single node cluster can be deployed using [Kind](https://kind.sigs.k8s.io/). This section describes the steps to get a single node kind cluster installed.

A Kind-based cluster should not be used in production. Please refer to [production best practices](https://kubernetes.io/docs/setup/production-environment/) before deploying a CNO Control Cluster for production use.

#### Install Docker Engine
If Docker is not installed, please [install](https://docs.docker.com/engine/install/) it first.

#### On Linux:
```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.16.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

#### On MacOS:
```
# for Intel Macs
[ $(uname -m) = x86_64 ]&& curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.16.0/kind-darwin-amd64
# for M1 / ARM Macs
[ $(uname -m) = arm64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.16.0/kind-darwin-arm64
chmod +x ./kind
mv ./kind /some-dir-in-your-PATH/kind
```

#### Create Cluster
```
kind create cluster --name control-cluster
kubectl cluster-info --context control-cluster
```

#### Install kubectl
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x kubectl
mv ./kubectl /usr/local/bin/kubectl
kubectl version --client"
```

#### Install Helm
Install Helm using [these](https://helm.sh/docs/intro/install/) istructions.

### Control Cluster Install Configuration
Use the following ```my_values.yaml``` when installing the CNO Helm Chart for the Control Cluster for release 0.9.1:

``` bash

cat > my_values.yaml << EOF
# Default values for netops-org-manager.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: quay.io/ckodev/netop-org-manager
  pullPolicy: Always
  # Overrides the image tag whose default is the chart appVersion.
  tag: "0.9.1.d04f56f"

## -- Specifies image details for netop-farbic-manager
fabricManagerImage:
  repository: quay.io/ckodev/netop-fabric-manager
  pullPolicy: Always
  # Overrides the image tag whose default is the chart appVersion.
  tag: "0.9.1.d04f56f"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

# -- Specifies whether to enable ValidatingWebhook
webhook:
  enable: true

## -- Specifies the log level. Can be one of ‘debug’, ‘info’, ‘error’, or any integer value > 0.
logLevel: "info"

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

resources: {}
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  # limits:
  #   cpu: 500m
  #   memory: 128Mi
  # requests:
  #   cpu: 10m
  #   memory: 64Mi

nodeSelector: {}

tolerations: []

affinity: {}

rbac:
  # Specifies whether RBAC resources should be created
  create: true

## -- Specifies `managerConfig`
managerConfig:
  controller_manager_config.yaml: |
    apiVersion: controller-runtime.sigs.k8s.io/v1alpha1
    kind: ControllerManagerConfig
    health:
      healthProbeBindAddress: :8083
    metrics:
      bindAddress: 127.0.0.1:8082
    webhook:
      port: 9443
    leaderElection:
      leaderElect: true
      resourceName: 2edab99a.

# -- Specifies whether to install a ArgoCD
argocd:
  enabled: true

# -- Specifies whether to install a Kubernetes Dashboard
kubernetes-dashboard:
  enabled: true
  rbac:
    clusterReadOnlyRole: true
EOF
```

### CNO Cleanup in Workload Cluster
[This script](scripts/cleanup-workload-cluster.sh) can be used to clean up CNO in the Workload Cluster (note the script will optionally allow you to delete CNI):

### CNO Auto-Restore in Control Cluster

The features provide functionality to pull the manifests from GitHub repo control/restore folder and apply it to the newly deployed control
cluster. This will help in auto-restore the control-cluster in case of failure or inaccessibility of the pre-existing cluster

#### Deploy using Helm:
To use auto-restore functionality follow below steps:
- Add extraEnv section to the template provided in the [Appendix](#control-cluster-install-configuration) to create the ```my_values.yaml```:
```
extraEnv:
- name: RESTORE_CLUSTER
  value: "true"
```

- Follow [Deploy using Helm](#314-deploy-using-helm) section to install
- If the <git-branch>/control/restore has CR files, it will automatically deploy those file with the same state as of previous cluster.
- Default RESTORE_CLUSTER is considered as false until and unless explicitly mentioned.

#### Multi-Cluster Transition:
To deploy the full cycle of creation in one cluster and restore into another cluster follow below steps:
##### Cluster-1:
- Create a new GitHub branch and provide secret credentials before helm-install. [Secret for GitHub access](#313-create-secret-for-github-access)
- Add extraEnv section to the template provided in the [Appendix](#control-cluster-install-configuration) to create the ```my_values.yaml```:
```
extraEnv:
- name: RESTORE_CLUSTER
  value: "true"
```

- Follow [Deploy using Helm](#314-deploy-using-helm) section to install
- Create all CRs (fabricinfra,clustergrouprofile, clusternetworkprofile, clusterprofile and clusterinfo) needed to deploy the workload cluster.
- Check in GitHub repo that control/restore has been populated with corresponding files

#### Cluster-2:
- Add extraEnv section to the template provided in the [Appendix](#control-cluster-install-configuration) to create the ```my_values.yaml```:
```
extraEnv:
- name: RESTORE_CLUSTER
  value: "true"
```

- Follow [Deploy using Helm](#314-deploy-using-helm) section to install
- If the <git-branch>/control/restore has CR files, it will automatically create those file with the same state as of previous cluster.
- Default RESTORE_CLUSTER is considered as false until and unless explicitly mentioned.

### Diagnostics:
### Scenario 1
In case a new cluster is created and the netop-org-manager pod goes into CrashloopBackoff
and throws error : `Failed to push file/remove - Cluster UUID does not match` when checking pod logs.

##### Observation
- Verify that the generated configmap `cluster-uuid`  with Data field uuid and file content in git-repo/(branch)/control/restore/cluster-uuid matches
- ```kubectl get configmap -n netop-manager cluster-uuid -o yaml```

##### Root Cause
- The generated configmap `cluster-uuid`  with Data field uuid and file content in git-repo/(branch)/control/restore/cluster-uuid not matches
- If different ,then some other control cluster is using the same GitHub branch and repository. 

##### Diagnostic Steps
- Identify which other control cluster is using the same GitHub repo/branch and operate that cluster.
- If the other control cluster is not identifiable or non-accessible, make use of unique repo/branch to avoid takeover.


### Scenario 2
In case of a working cluster and the netop-org-manager pod goes into CrashloopBackoff
and throws error : `Failed to push/remove file - Cluster UUID does not match` when checking pod logs.

##### Observation
- Verify that the generated configmap `cluster-uuid`  with Data field uuid and file content in git-repo/(branch)/control/restore/cluster-uuid matches
- ```kubectl get configmap -n netop-manager cluster-uuid -o yaml```

##### Root Cause
- The generated configmap `cluster-uuid`  with Data field uuid and file content in git-repo/(branch)/control/restore/cluster-uuid not matches
- If different ,then some other control cluster is using the same GitHub branch and repository.

##### Diagnostic Steps
- Identify which other control cluster is using the same GitHub repo/branch and remove that cluster.
- If the other control cluster is not identifiable or non-accessible, ensure using unique repo/branch to avoid takeover.


### Scenario 3
In case restore functionality is configured and still do not see files being pushed/removed from GitHub Repository.

##### Observation
- Verify that the `RESTORE_CLUSTER` environment variable is `true` in netop-org-deployment


##### Root Cause
- The `RESTORE_CLUSTER` environment variable is marked `false` or not present in netop-org-deployment

##### Diagnostic Steps
- Edit the netop-org-manager deployment in netop-manager namespace
-```kubectl edit deployment -n netop-manager        netop-org-manager-netop-org-manager```
- Add the `RESTORE_CLUSTER` env
```
- name: RESTORE_CLUSTER
-  value: "true"
```
-This will restart the netop-org-manager pod with auto-restore enabled

### CNO Shadow Resource in Control Cluster

The features offer a backup mechanism for the generated acc_provision_input and netop-manager deployment manifests, 
ensuring their preservation in the event of APIC unprovisioning failure.

##### Exisiting Condition

In the current situation, when there is a failure in the process of unprovisioning, the user faces a challenge in 
identifying the specific factors that contributed to the failure. To troubleshoot, the user has limited options: either manually accessing the pod and examining the generated manifests, 
or debugging the CRs (Custom Resources) involved.

##### Requirement
Given the current state of affairs, there is a need to establish a convenient method for referencing and accessing failed or incorrect manifests.

##### Enhancements
Taking into account the aforementioned requirement, a proposed solution is to create a "shadowresource" Custom Resource (CR) using netop-fabric-manager. 
This CR would capture a snapshot of the currently generated manifests during provisioning. 
In the event of a failure in APIC unprovisioning, as indicated by the netop-fabric-manager pod logs, users can refer to these shadowresources for troubleshooting purposes.

##### Diagnostic Steps
###### On Existing Cluster
Run the following command to retrieve the shadowresource for a specific cluster profile in the netop-manager namespace:

- ```kubectl get shadowresources -n netop-manager <cluserinfoname>```

Note that the shadowresource will have the same name as the clusterinfo.

###### On New Cluster
If the user ends up deploying a new cluster, they can still reference previously generated shadow resources by accessing the Git repository at the following URL
```
https://github.com/<REPO>/<DIR>/tree/<BRANCH>/workload/shadowresource
```
Make sure to replace <REPO>, <DIR>, and <BRANCH> with the appropriate values corresponding to the Git repository.

