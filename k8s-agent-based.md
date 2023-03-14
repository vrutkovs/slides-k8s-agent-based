## Agent-based k8s installation
or
#### How I stopped worrying and learned to install k8s clusters
<br>
<br>
<br>
Vadim Rutkovsky

vrutkovs@redhat.com

---
### `whoami`
Principal software engineer from Belarus living in Czech Republic.

Working for Red Hat in the OpenShift department.

Note:

My job is to overlook cluster lifecycle, helping customers install clusters of various configuration, extend, upgrade and manage at scale.

---
### Problem statement
Cluster lifecycle FAQ:

* What happens during cluster installation? What if it goes sideways?

Note:

Our customers need to know if installation method is flexible enough to work on their infrastructure. So we need a through description of the installation process.

We also need a way to collect necessary debug information when installation fails.

---
### Problem statement
Cluster lifecycle FAQ:

* Cluster got installed? Now repeat it a hundred times

Note:

Customers may want to install multiple clusters, so the installation method has to be resilient. It also has to have a defined set of user inputs so that it would be reproducible.

---
### Problem statement

Cluster lifecycle FAQ:

* It seems easy to install a k8s cluster, can it be a self-service?

Note:

We want to make installation boring, so that we could hand it off cluster installation to developers and offer it as a service

---
### Problem statement

Cluster lifecycle FAQ:

* Can I use GitOps for this?

Note:

GitOps offers a way of tracking who did what and allows change review before they have happened, which is invaluable for cluster lifecycle. This allows us to keep a history of cluster changes for the whole team managing the cluster.

---
### The installer dilemma

Multiple possible ways - `kops`, `kubeadm`, `kubespray` etc.
These are balancing between two extremes:

---
### The installer dilemma

* Too generic installer

Note:
the method is too generic as you have to come up with additional tools and its hard to troubleshoot heavily customized installations (especially on baremetal). For instance, `kubeadm` covers just the k8s part, not host preparation part, meaning OS bits may influence installation process and need to be accounted for.

---
### The installer dilemma

* Too specific installer

Note:
the method is too specific for particular provider - installs all the necessary infra in the cloud (i.e. via Terraform) which makes it harder to customize - also, it most cases its complicated to apply on baremetal installs.

---
### What? Another method?

Yes.

The focus here is Baremetal, as it requires careful configuration, a lot of customization and makes troubleshooting complicated.

Note:
we needed an installation method which tries to find the middle ground between two extremes, focusing on user-provided infrastructure, which is common on bare-metal installations.

---
### What? Another method?

Requirements:

* Hide installation complexity
* Flexible infra requirements
* Help with troubleshooting

Note:
The method would combine the benefits of the two approaches:
* automate boring parts
* customizable
* able to collect debug information automatically

This leads us to an idea of Assisted Installer - a tool which helps you identify if your cluster settings are valid and nodes fulfil the requirements


---
### Agent-based method
Boot into a Live ISO and receive host information

![host-overview](imgs/host-info.png)

Note:
Instead of provisioning the machine with real OS lets first collect information about available hosts.

In order to do that we need to boot every machine via a Live ISO and have a special agent there which fetches necessary information. This service would send it to the DB, which can be queried by a service (say, web-based) and a pretty UI on top of it.

This would help us solve several problems:

* Before cluster installation we filter out broken machines - if it can't run the agent it won't run kubelet
* Have a full overview of available hosts
* Establish a communication channel with the machine without SSH

---
### It Takes Two to Tango

Bidirectional channel - the host can send logs back, the service can send commands to run on the host

Note:
A communication channel is important, as now the hosted service can now send and receive files from the host, meaning:

* Necessary files like configuration, certificates etc can be sent on host
* The host can return error messages or logs if particular operation fails

---
### Fully Automated Gay Space Luxury Installation

API is 💪

![api](imgs/api.png)

Note:
Since we rely on hosted service, we might as well run the installer there and send artifacts on the service.
This allows us to interact with the service without a UI. As a result, the hosted service can be API-driven, which enabled careful tweaking and improved control over the installation process.

---
### Pretty is a feature

Since the service has an API, it can be visualized - using a Web UI

![validations](imgs/validations.png)

Note:
Cute UI is always a thing which makes UX easy for newcomers. Users would spend less time doing boring things and get notified about critical problems on the way.

---
### Validations
Find problems before installation begins, such as:
* Nodes unable to communicate with each other
* Slow disks for control plane nodes

![failing-validation](imgs/failing-validation.png)

Note:
One of the distinctive features of this installation method is the ability to validate configuration before installation has stared.

Validations can highlight critical problems on the hosts or cluster configuration. One example may be obvious things like k8s network misconfigurations or more subtle issues - like time difference on the hosts, which may result in some certificates not being valid yet from the host point of view.

---
### Isn't it Ironic?
IPMI / RedFish etc -> Declarative automatic discovery and provisioning

![bmc](imgs/bmc.png)

Note:
Assisted Service also includes component from OpenStack called Ironic, which unlocks the ability to discover and configure machines via IPMI, RedFish etc. This component helps us automatically powerr on the machine, provision it with discovery ISO, collect and validate host details and proceed with installation - all without user interaction.

---
### Live installation progress
UI shows installation progress:
* Control plane preparation
* Nodes joining the cluster
* Finalization

![installation-progress](imgs/installation-progress.png)

Note:
A lot of things may go wrong during cluster installation, so the service oversees the installation progress and its phases. Users need to be notified about problems in case it didn't transition to the expected state.

---
### Why not Cluster API?

Validations would not stop the broken cluster from installing.

Related work:

https://github.com/vmware-tanzu/cluster-api-provider-bringyourownhost

https://github.com/openshift/cluster-api-provider-agent

Note:
Cluster API is a great tool to spin up new cluster, but its features were insufficient. The biggest problem here is that the installation has already started, so host validations cannot prevent the misconfigured cluster from starting the installation. However, if you want to follow that route Tanzu folks have a provider which uses similar idea.

It is also fairly easy to build a Cluster API provider using Openshift Assisted Installer agents too.
Openshift Assisted Service is using a custom Cluster API provider to make use of discovery ISO agents.

---
### But how do I ...?

There are quite a lot of other problems we'd need to cover:

---
### But how do I ...?

#### Networks

OpenShift uses `NMState` operator to prepare NetworkManager configuration to setup network parameters on installation.

Note:
Networking is never easy. In Openshift Assisted Service we're relying on NetworkManager to configure it for us. NetworkManager configuration can be generated from manifests using NMState operator.

During discovery phase users may need to manually ssh on the hosts to establish agent-service communication channel.

---
### But how do I ...?

#### Customized node configuration

`rpm-ostree` - https://rpm-ostree.readthedocs.io/en/stable/

`Ignition` - https://coreos.github.io/ignition/

Note:
Cluster installation needs to include OS provisioning for better results. In Openshift we rely on two technologies:
* `rpm-ostree` to write OS images on disk and manage atomic upgrades

* Ignition is used in OpenShift to create necessary files / systemd services to customize node contents.

After cluster installation both of these technologies are used to control files on disk in Machine Config Operator.

---
### But how do I ...?

#### Can use another k8s distro?

"Vanilla" Kubernetes is a too broad of a term, needs to be narrowed down.

Note:
Vanilla k8s comes in different sizes, so in order to make a viable service it needs to be narrowed down to a particular scope - focusing on NetworkManager, selecting an OS which can be provisioned with Ignition etc. etc.

---
### Lies, damned lies and statistics
Collect cluster installation success rate, find problems sooner etc.

![grafana](imgs/grafana.png)

Note:

Since the agent is reporting cluster status to the central service we as service providers can easily spot regressions in the service configuration by analyzing pass rate for installed clusters. This helps us maintain 75% of installation success rate

---
### Hello operator

Since its a webservice, it can run in k8s. Moreover, it can also fetch input from k8s manifests and start installation in a declarative hands-off mode.

Note:
We also offer this service as a part of Advanced Kubernetes Management product and in order to make API accessible to the power users, its being served as a k8s operator. This allows us to define cluster properties as k8s custom resources

---
### Of course it supports GitOps

#### Zero Touch Provisioning

Cluster definition stored in Git, applied by Flux/ArgoCD, reconciled by the operator

Note:
Since we can now start cluster installation via a handful of k8s manifests, these can be stored in a gitrepo and applied automatically - GitOps methods.

In OpenShift speak this method is called "Zero Touch Provisioning"

---
### Screenshot: Cluster details

![cluster-details](imgs/cluster-details.png)


Note:
Let's do a small screenshot tour to highlight the features of this approach. First we start with high-level cluster details - name, base domain and cluster version

---
### Screenshot: Discovery ISO

![discovery-iso](imgs/discovery-iso.png)


Note:
Now we need to generate the ISO to fetch information about machines. ISOs come in two different flavors - one is "minimal ISO" which only has kernel and rootfs is being fetched from the service. Its mostly useful in virtualized environments where network is usually DHCP-based and pre-configured.
The other one is "full ISO" which includes the rootfs.

Agent may not start for various reasons, so we bake in an SSH public key so that agent startup could be diagnosed using traditional method.

---
### Screenshot: Discovered hosts

![hosts-list](imgs/hosts-list.png)

Note:
Once the discovery ISO boots, agent is started and can communicate with the service we can now review hosts details. The hosts can already get roles assigned - like "control plane" or "worker". Some validation already run here - i.e. we don't allow to run control plane hosts on HDD or all hostname are distinct. We also check that ensure time sync is enabled on all hosts so that later on some certificates would not be mistakenly considered invalid.

---
### Screenshot: Network Settings

![network-settings](imgs/network-settings.png)

Note:
Next we need to configure the networking preferences like IPv4-only or dual-stack, CIDR ranges and so on.

---
### Screenshot: Installation progress

![installation-progress](imgs/installation-progress.png)

Note:

When installation has started the service would show us current phase each host is at.

---
### tada.wav
![installation-complete](imgs/installation-complete.png)

Note:

After all installation procedures are done the service is giving cluster console URL along with pre-generated credentials to get started.

---
### Operator: Network Configuration

```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: NMStateConfig
metadata:
  name: assisted-deployment-nmstate-lab-spoke
  labels:
    cluster-name: nmstate-lab-spoke
spec:
  interfaces:
    - name: "eth0"
      macAddress: "02:00:00:80:12:14"
    - name: "eth1"
      macAddress: "02:00:00:80:12:15"
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
```

Note:

Lets take a peek into more complicated things. As I previosly mentioned the networking configuration is also defined via manifests. In our implementation its controlled by NMState operator. Here's an example of setting up a bond for two interfaces on the host.

---
### Operator: Provision via RedFish

```yaml
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: lab-agent1
  namespace: open-cluster-management
  labels:
    infraenvs.agent-install.openshift.io: "lab-env"
spec:
  online: true
  bmc:
    address: redfish-virtualmedia+http://[2620:52:0:1302::100]:8000/redfish/v1/Systems/foo
    credentialsName: bmc-secret1
    disableCertificateVerification: true
  bootMACAddress: ee:bb:aa:ee:1e:1a
  automatedCleaningMode: disabled
```


Note:

Baremetal installation can be automated thanks to OpenStack Ironic. For each host we'd need to define an URL to baremetal control interface and a link to its credentials. During installation, Ironic would power on the host, run discovery ISO in Live ISO mode, collect host details and when installation starts it would write the OS image on the selected disk and control host power state, rebooting it when necessary. This Ironic would later on the cluster itself, so you get the same interface during installation and upgrade too.

---
### Epilogue

Agent-based installation gives several benefits over traditional ways of installing a cluster:

* User input validation the cluster settings before writing anything to disk
* Provides an API which unlocks self-service and automation
* Combines infra flexibility and UX

---
### Thanks + Questions

Find me at https://vrutkovs.eu

https://vrutkovs.github.io/slides-k8s-agent-based/


---
### Links

More information:

* https://github.com/openshift/assisted-service
* https://cloud.redhat.com/blog/how-to-use-the-openshift-assisted-installer
* https://vrutkovs.eu/posts/okd-disconnected-assisted/
* https://cloud.redhat.com/blog/telco-5g-zero-touch-provisioning-ztp
