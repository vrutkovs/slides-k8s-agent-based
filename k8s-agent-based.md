## Agent-based k8s installation
### or How I stopped worrying and learned to install multiple clusters

Vadim Rutkovsky

vrutkovs@redhat.com

---

### `whoami`

I am a principal software engineer from Belarus living in Czech Republic.

Working for Red Hat in OpenShift department.

My job is to overlook cluster lifecycle, helping customers install clusters of various configuration, extend, upgrade and manage at scale.

---
### Problem statement

Here are most common questions about cluster installation we get from customers:

* What happens during cluster installation?
* What if it goes sideways?
* Cluster got installed? Good, now do this hundred times more on different configurations
* It seems easy to install a k8s cluster, can it be a self-service?
* I heard GitOps is pure magic, can I use it install a cluster?

Lets look into all these points

---
### How does cluster generally get installed

Multiple possible ways - `kops`, `kubeadm`, `kubespray` etc.

These are balancing between two extremes here:

* either the method is too generic that its hard to troubleshoot on real configuration (especially baremetal)
* or the method installs all the necessary infra in the cloud (i.e. via Terraform) which makes it harder to apply on baremetal installs.

---
### What? Another method?

Yes. Baremetal here is the focus, as it requires careful configuration, a lot of customization and makes troubleshooting complicated.

General install methods like `kubeadm` just tell you the necessary part need to be fulfilled, instead of actions to perform. Its also not helpful during troubleshooting as it doesn't come with a debug tool.

This leads us to an idea of Assisted Installer - a tool which helps you identify if your cluster settings are valid and nodes fulfil the requirements

---
### Agent-based method

Instead of booting into real OS and getting started with kubelet and certificates lets first collect information about available hosts. In order to do that we need to boot every machine via a Live ISO and have a special agent there which fetches necessary information. This service would send it to the DB, which can be queried by a service (say, web-based) and a pretty UI on top of it.

This is a basic description of OpenShift Assisted Service.

![host-overview](imgs/assisted-service-host-overview.png)

This would help us solve several problems:

* Before cluster installation we filter out broken machines - if it can't run the agent it won't run kubelet
* Have a full overview of available hosts
* Establish a communication channel with the machine without SSH

---
### It Takes Two To Tango

A communication channel is important, as now the hosted service can now send and receive files from the host, meaning:

* Necessary files like configuration, certificates etc can be sent on host
* The host can return error messages or logs if particular operation fails

---
### Fully Automated Gay Space Luxury Installation

Since we rely on hosted service, we might as well run the installer there and send artifacts on the service. This allows us limit the inputs from the user and improves reproducibility. As a result, the hosted service can be API-driven, which enabled careful tweaking and improved control over the installation process.

---
### Pretty is a feature

If the service has an API it can be visualized - using Web UI - and authenticated, so that the end-users would not mix their own clusters.

![cluster-list](imgs/assisted-service-cluster-list.png)

---
### Validations

check CIDR, 

---
### Isn't it Ironic?

Assisted Service also includes OpenStack Ironic, which unlocks the ability to discover and configure machines via IPMI, RedFish etc.

---
### Installation progress control

see if all nodes have joined the cluster

---
### Day 2 operations

Add more nodes to the cluster

---
### But how do I ...?

There are quite a lot of other problems we'd need to cover, most of them boil down to:

Networks - OpenShift uses `NMState` operator to prepare NetworkManager configuration to setup network parameters on installation.

Custom configuration - `Ignition` is used in OpenShift to create necessary files / systemd services to customize node contents. Assisted Service may also receive additional k8s manifests we want to be applied during installation (i.e. different CNI, ArgoCD installation and subscription etc.)

---
### How are my clusters doing?

Collect cluster installation success rate, find problems sooner etc.

---
### Hello operator

Since its a webservice, it can run in k8s. Moreover, it can also fetch input from k8s manifests and start installation in a hands-off mode.

---
### Of course it supports GitOps

Since we can now start cluster installation via a handful of k8s manifests, these can be stored in a gitrepo and applied automatically - GitOps methods.

In OpenShift speak this method is called "Zero Touch Provisioning"

---
### Example: ZTP hub configuration p. 1

```yaml
apiVersion: hive.openshift.io/v1
kind: ClusterImageSet
metadata:
  name: openshift-v4.8.0
  namespace: open-cluster-management
spec:
  releaseImage: quay.io/openshift-release-dev/ocp-release:4.8.0-fc.8-x86_64
```
This manifest describes which OpenShift release we want to install.

The release image has references to all things cluster will need - kube-apiserver image, OpenShift-specific operators and even the machine OS image

---
### Example: ZTP hub configuration p. 2

```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: AgentServiceConfig
metadata:
  name: agent
  namespace: open-cluster-management
spec:
  ...
  ### This is a ConfigMap that has credentials to local mirror
  mirrorRegistryRef:
    name: "mirror-mirror-on-the-wall"
  ### This describes which ISO is used a base for Discovery ISO
  osImages:
    - openshiftVersion: "4.8"
      version: "48.84.202106070419-0"
      url: "https://releases-rhcos-art.cloud.privileged.psi.redhat.com/storage/releases/rhcos-4.8/48.84.202106070419-0/x86_64/rhcos-48.84.202106070419-0-live.x86_64.iso"
      rootFSUrl: "https://releases-rhcos-art.cloud.privileged.psi.redhat.com/storage/releases/rhcos-4.8/48.84.202106070419-0/x86_64/rhcos-48.84.202106070419-0-live-rootfs.x86_64.img"
```
This manifest specifies agent settings.

---
### Example: ZTP hub configuration p. 3

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: assisted-deployment-ssh-private-key
  namespace: open-cluster-management
stringData:
  ssh-privatekey: |-
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
type: Opaque
```
Things may go wrong and we may need to SSH on the host to start the agent. This key is not necessary for cluster installation.

---
### Example: ZTP hub configuration p. 4

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: assisted-deployment-pull-secret
  namespace: open-cluster-management
stringData:
  .dockerconfigjson: '{"auths":{"registry.ci.openshift.org":{"auth":"MYAUTHSTRING"},"quay.io":{"auth":"ANOTHERAUHTSTRING==","mirror-mirror-on-the-wall:5000":{"auth":"ANOTHERAUTH=","email":"vadim@vrutkovs.eu"}}}'
```
Images may require authentication to pull

---
### Example: ZTP hub configuration p. 5

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: assisted-deployment-pull-secret
  namespace: open-cluster-management
stringData:
  .dockerconfigjson: '{"auths":{"registry.ci.openshift.org":{"auth":"MYAUTHSTRING"},"quay.io":{"auth":"ANOTHERAUHTSTRING==","mirror-mirror-on-the-wall:5000":{"auth":"ANOTHERAUTH=","email":"vadim@vrutkovs.eu"}}}'
```
Images may require authentication to pull


---
### Example: target cluster configuration p. 1

The following manifests describes what kind of cluster do we want
```yaml
apiVersion: extensions.hive.openshift.io/v1beta1
kind: AgentClusterInstall
metadata:
  name: lab-cluster-aci
  namespace: open-cluster-management
spec:
  clusterDeploymentRef:
    name: lab-cluster
  imageSetRef:
    name: openshift-v4.8.0
  networking:
    clusterNetwork:
      - cidr: "fd01::/48"
        hostPrefix: 64
    serviceNetwork:
      - "fd02::/112"
    machineNetwork:
      - cidr: "2620:52:0:1302::/64"
  provisionRequirements:
    controlPlaneAgents: 1
  sshPublicKey: "ssh-rsa adasdlkasjdlklaskdjadoipjasdoiasj root@xxxxXXXXxxx"
```
That would be OpenShift 4.8, single control plane node with specified network settings.

---
### Example: target cluster configuration p. 2

ClusterDeployment describes which nodes should be included in the cluster and its DNS settings.
```yaml
apiVersion: hive.openshift.io/v1
kind: ClusterDeployment
metadata:
  name: lab-cluster
  namespace: open-cluster-management
spec:
  baseDomain: vrutkovs.eu
  clusterName: okd
  controlPlaneConfig:
    servingCertificates: {}
  installed: false
  clusterInstallRef:
    group: extensions.hive.openshift.io
    kind: AgentClusterInstall
    name: lab-cluster-aci
    version: v1beta1
  platform:
    agentBareMetal:
      agentSelector:
        matchLabels:
          size: "large"
  pullSecretRef:
    name: assisted-deployment-pull-secret
```

---
### Example: target cluster configuration p. 3

NMState describes network configuration of cluster nodes:
```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: NMStateConfig
metadata:
  name: assisted-deployment-nmstate-lab-spoke
  labels:
    cluster-name: nmstate-lab-spoke
spec:
  config:
    interfaces:
      - name: bond99
        type: bond
        state: up
        ipv6:
          address:
          - ip:2620:52:0:1302::100
            prefix-length: 64
          enabled: true
        link-aggregation:
          mode: balance-rr
          options:
            miimon: '140'
          slaves:
          - eth0
          - eth1
  interfaces:
    - name: "eth0"
      macAddress: "02:00:00:80:12:14"
    - name: "eth1"
      macAddress: "02:00:00:80:12:15"
```

---
### p 4

InfraEnv describes configuration for a group of hosts:
* custom Ignition setup to add `/etc/someconfig` file on each node
* NMState config created in the previous section

```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: InfraEnv
metadata:
  name: lab-env
  namespace: open-cluster-management
spec:
  clusterRef:
    name: lab-cluster
    namespace: open-cluster-management
  sshAuthorizedKey: "ssh-rsa adasdlkasjdlklaskdjadoipjasdoiasj root@xxxxXXXXxxx"
  agentLabelSelector:
    matchLabels:
      size: "large"
  pullSecretRef:
    name: assisted-deployment-pull-secret
  ignitionConfigOverride: '{"ignition": {"version": "3.1.0"}, "storage": {"files": [{"path": "/etc/someconfig", "contents": {"source": "data:text/plain;base64,aGVscGltdHJhcHBlZGluYXN3YWdnZXJzcGVj"}}]}}'
  nmStateConfigLabelSelector:
    matchLabels:
      cluster-name: nmstate-lab-spoke
```

---
### p 5

`BareMetalHost` uses `RedFish` protocol to install a discovery ISO on the baremetal host.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bmc-secret1
  namespace: open-cluster-management
data:
  password: YWxrbm9wZmxlcgo=
  username: YW1vcmdhbnQK
type: Opaque
---
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: lab-agent1
  namespace: open-cluster-management
  labels:
    infraenvs.agent-install.openshift.io: "lab-env"
  annotations:
    inspect.metal3.io: disabled
spec:
  online: true
  bmc:
    address: redfish-virtualmedia+http://[2620:52:0:1302::d7c]:8000/redfish/v1/Systems/3e6f03bb-2301-49c9-a562-ad488dca513c
    credentialsName: bmc-secret1
    disableCertificateVerification: true
  bootMACAddress: ee:bb:aa:ee:1e:1a
  automatedCleaningMode: disabled
```

---
### p6 

If IPMI and similar are not available the user can create boot nodes using ISO from `oc get infraenv lab-env -o jsonpath={.status.isoDownloadURL}`. Once the agent has booted, the controller would create necessary `Agent` CRs and start the installation.

---
### p7 - Start the engines

Now once we're all set, time to signal the controller that we want the cluster install to begin:

```yaml
apiVersion: hive.openshift.io/v1
kind: ClusterDeployment
metadata:
  name: lab-cluster
  namespace: open-cluster-management
spec:
  installed: tru
```

and watch `.status.conditions` of this object.

---
### Bottom line

Agent-based installation gives several benefits over traditional ways of installing a cluster:

* Validating the cluster setting before writing anything to disk
* Overview of the progress
* Provides API, which unlocks self-service and automation

---
## Questions?

https://vrutkovs.github.io/slides-okd-installer-screenshots
