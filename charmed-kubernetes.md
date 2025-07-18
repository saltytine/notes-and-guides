# charmed kubernetes

see official documentation [here](https://documentation.ubuntu.com/canonical-kubernetes/latest/).  
charmed kubernetes is canonical’s enterprise kubernetes distribution, designed to provide a model-driven, fully managed container orchestration platform on ubuntu.  it uses juju “charms” to model and operate each kubernetes component, enabling automated deployment, upgrades, and scaling of clusters on any infrastructure.  in practice, a juju controller and model define the cluster topology – for example, an etcd cluster charm, multiple kubernetes-control-plane units, and kubernetes-worker units – with charms for network, storage, and other services all related together.  this approach makes charmed k8s highly modular and adaptable to environments from bare-metal to public cloud.  the architecture diagram below illustrates this model-driven design: juju sits at the center orchestrating container runtimes (something like containerd or kata), cnis (calico/flannel), storage (ceph), and observability (prometheus, grafana, loki, etc.) in a unified stack.

```
                          +------------------+
                          |  juju controller |
                          +--------+---------+
                                   |
                          +--------v---------+
                          |   juju model     |
                          +--------+---------+
                                   |
        +--------------------------+---------------------------+
        |                          |                           |
        v                          v                           v
+-----------------+       +--------------------+      +--------------------+
| kubernetes-ctl  |       |   etcd (cluster)   |      |  kubernetes-worker |
| (control-plane) |<----->| (tls, quorum 3/5)  |<---->|   (kubelet, cni)   |
+-----------------+       +--------------------+      +--------------------+
        |                          |                           |
        v                          v                           v
+----------------+     +--------------------+       +---------------------+
|  containerd    |     |   easyrsa/vault    |       |     cni plugin      |
| (or kata opt.) |     |    (tls certs)     |       | (calico/flannel/..) |
+----------------+     +--------------------+       +---------------------+

                         +--------------------+
                         |   observability    |
                         | (prom, loki, etc.) |
                         +--------------------+

```

the design philosophy is declarative and “day-2” automated – charmed kubernetes aims to manage the entire lifecycle (deployment, updates, scaling, failure recovery) without manual intervention.  by default every core kubernetes service is delivered as a juju charm.  for example, the kubernetes-control-plane charm runs the api server, controller-manager, and scheduler; the kubernetes-worker charm runs kubelet, kube-proxy and a container runtime; and the etcd charm runs a tls-enabled etcd cluster.  this means updates (via juju “charm refresh” or snap refresh) and configuration changes (via juju model configs) can be applied uniformly to all units.  because the bundle uses ubuntu snaps for kubernetes binaries, os and security patches are delivered automatically, ensuring cncf-certified kubernetes with up-to-date fixes.  charmed k8s also supports advanced infrastructure features: like full oci compatibility (containerd and docker), pci passthrough (gpu/fpga/sr-iov), and multi-arch (x86/arm) for iot/edge use.

## components

charmed k8s breaks kubernetes down into juju-managed charms.  core components and their deployment are:

* kubernetes control plane (kubernetes-control-plane charm): this charm installs and configures the api server, controller-manager, and scheduler on master nodes.  it is deployed with multiple units for ha, and is usually fronted by a load-balancer or vip (see ha section).  the control-plane charm also relates to easyrsa or vault for tls certs, and to other charms like lb or ingress as needed.
* etcd (etcd charm): etcd runs as a separate charm providing a tls-terminated clustered key-value store.  the etcd charm typically runs 3 or 5 units for quorum and is automatically bootstrapped via juju.  it stores all kubernetes state and supports snapshots via juju actions.  the charm uses the easyrsa ca for its own certificates (see security).
* kubernetes worker (kubernetes-worker charm): worker nodes run the kubelet and kube-proxy.  this charm also installs the container runtime (by relating in a runtime charm) and the chosen cni (flannel/calico/etc.) as subordinates.  workers register themselves with the control plane automatically.
* container runtime: charmed k8s primarily uses containerd (via the containerd subordinate charm) as the runtime (support for docker was removed in v1.24).  the containerd charm can be configured (for example registry-mirror, registry-auth, or gpu support) and related to kubernetes-worker and kubernetes-control-plane charms.  optionally, kata containers can be enabled by deploying the kata charm alongside containerd; since v1.16, kata provides vm isolation for untrusted pods.
* networking (cni): by default charmed k8s deploys calico for pod networking.  alternative cnis (flannel, canal, cilium, tigera, etc.) can be deployed via juju overlays or by relating their charms to the workers.  for example, the flannel or canal charms can be related to kubernetes-worker for layer-3 connectivity.  metallb can be used on bare-metal to provide external ips to services.
* ingress/load balancing: the kubeapi-load-balancer charm (nginx) or cloud lbs can provide a front-end for the api servers in ha setups.  similarly, ingress controllers (nginx, contour, etc.) and load balancers can be deployed via charm.  charmed k8s bundles often use keepalived or metallb to implement virtual ips for control-plane ha.
* storage: charmed k8s can integrate with ceph or other storage backends.  in an overlay, operators deploy ceph-mon and ceph-osd charms to form a ceph cluster, then deploy the ceph-csi charm to provide rbd/cephfs storage classes.  persistentvolumes are then provisioned via standard pvcs.  (alternatively, cloud volumes, or local volumes can be used.)
* others: many complementary services have charms that integrate seamlessly.  for tls and cert management, the easyrsa charm acts as a ca for all other charms.  for logging/monitoring, there are prometheus2, grafana, loki, alertmanager, filebeat, etc. (see observability).  for compliance/security, the cis-benchmark action in the control-plane/worker/etcd charms can evaluate cis compliance.  operators (charms) for apps (databases, caches, etc.) can be related to the cluster via juju or deployed in the cluster itself.

## juju integration

charmed kubernetes is built on juju’s model-driven operations paradigm.  the juju controller orchestrates the lifecycle of every component via charms and relations.  a juju model encapsulates one kubernetes cluster (with its controller, units, and relationships).  the juju controller and models are portable – you can create models in any cloud, bare-metal, or even on-premises via maas.  common charms used in the default charmed k8s bundle include easyrsa, etcd, kubernetes-control-plane, kubernetes-worker, kubeapi-load-balancer, plus subordinate charms for cni (flannel, calico, etc.), storage (ceph-csi, rook, etc.), and compute (docker, containerd, kata).  each charm declares the interfaces it provides/requires.  for example, kubernetes-control-plane requires a tls cert provider and provides endpoints for juju-info (used for monitoring) and kubecontrolplane relations.  juju relations automatically exchange configuration: for example, the kubernetes-control-plane:prometheus relation to prometheus2:manual-jobs ensures prometheus scrapes the api server.  lifecycle hooks in each charm handle events: on config-changed a charm reconfigures the underlying component, on upgrade-charm it performs in-place upgrades, etc.  this means adding a machine and running juju add-unit will deploy the charm, configure the service, and join it to the cluster.  as ubuntu’s documentation notes, “every service is driven by a charmed operator”, enabling automated integration and updates.

juju also provides overlays and bundles.  bundles specify a set of charms and relations for a complete cluster.  overlays can modify the bundle (to switch cni or add monitoring).  for example, a monitoring overlay relates prometheus, grafana, telegraf to the control-plane and workers as shown in the documentation.  juju’s model abstraction enables multi-cluster topologies and multi-cloud deployment via juju controllers that span clouds.  in short, charmed k8s leverages juju’s whole-application orchestration model: engineers simply declare the desired topology and juju charms take care of bootstrapping, configuration, and day-2 tasks.

## deployment and lifecycle management

charmed kubernetes clusters are typically deployed via juju bundles on a juju model.  by default, the charmed-kubernetes bundle will deploy a single control-plane unit (plus etcd and easyrsa) and one or more workers.  for production, an ha topology is used (see below).  the deployment sequence (as in a bundle) is generally: bootstrap juju controller, add target machines (like with juju add-machine), then juju deploy charmed-kubernetes --overlay <custom.yaml>.  juju will provision machines (or spin up vms), install charms and snaps, and relate them.  for example, in a minimal core setup juju would install one unit each of kubernetes-control-plane, etcd, and easyrsa on a machine, and one kubernetes-worker on another.  the control-plane registers with etcd (via the tls-certificates interface), and the worker registers with the api server.  the installer then provides a kubeconfig to allow kubectl access.
```
example deployment:
-----------------
1. juju bootstrap <cloud>
2. juju add-model <model-name>
3. juju add-machine (optional)
4. juju deploy charmed-kubernetes --overlay <custom.yaml>

result:
---------------------
[etcd] <--> [kubernetes-control-plane] <--> [kubernetes-worker]
   |                   |                            |
   v                   v                            v
[easyrsa]        [kubeapi-load-balancer]       [containerd + CNI]

```

### high availability

charmed k8s supports ha out-of-the-box.  the documentation recommends deploying multiple control-plane units with a front-end load balancer or vip.  for example, one common pattern is to add 3 kubernetes-control-plane units, and deploy the kubeapi-load-balancer (nginx) charm with juju deploy kubeapi-load-balancer.  then relate it to all control-plane units; the workers use the lb’s ip to reach the api.  alternatively on bare-metal, the keepalived charm can float a virtual ip across the masters.  regardless of method, the goal is that if one master fails, the others (behind the lb/vip) keep the cluster alive.  etcd itself should run in an odd-numbered quorum (3 or 5 units) so it tolerates node failures.  (by default the etcd charm is deployed with 3 units, which juju co-locates or can distribute on separate machines.)  worker nodes can also be scaled (via juju add-unit kubernetes-worker) without downtime.  ingress traffic for workloads can be ha’d using metallb or keepalived plus any ingress controller.

### upgrades

charmed kubernetes automates patch upgrades.  that is, point releases within the same minor version are automatically applied by refreshing the underlying snaps and charms.  the operator need only ensure the controller and model are updated.  minor version upgrades (like 1.28 to 1.29) require a managed process: the [upgrade documentation](https://ubuntu.com/kubernetes/charmed-k8s/docs/upgrading) instructs operators to follow specific steps, usually by revising the charms to a new channel and carefully upgrading one service at a time.  in all cases, backups are strongly recommended beforehand: make etcd snapshots and back up any pv data:contentreference[oaicite:27]{index=27}:contentreference[oaicite:28]{index=28}.  after an upgrade, juju status\ should show all units “active” with the new version.  because the charm developers control the packaged kubernetes, deprecated apis or special migrations are handled in charm hooks when possible.  operators only need to coordinate roleout (cordoning nodes, draining workloads) as in any k8s upgrade.

### scaling and maintenance

scaling is simple: add units.  like juju add-unit kubernetes-worker adds another worker vm or container and automatically joins it to the cluster.  control-plane scaling (adding masters) is possible but must be paired with ha network setup as above.  juju also supports juju upgrade-charm which upgrades individual charms in-place; this is used for official charm upgrades or to switch from easyrsa to vault, etc.  for zero-downtime maintenance, juju’s model-driven nature allows rolling upgrades: for example, one can upgrade one control-plane unit at a time or do canary tests.

operators should monitor juju’s built-in automatic updates: the kubernetes snaps are updated on the machines, so it’s important to control the snap refresh schedule if needed (see “snap refresh settings” in docs).  by default ubuntu will apply security patches rapidly to all snaps, providing a “zeroops” experience.  the juju charm metadata also allows setting an upstream channel for kubernetes versions.  overall, charmed k8s aims for a mostly unattended lifecycle, with the operator intervening mainly for major upgrades or custom config changes.

### backup and restore

the etcd database holds the cluster state (objects, secrets, etc), so it must be backed up.  charmed k8s exposes etcd’s native snapshotting via juju actions on the etcd units.  the [backup guide](https://ubuntu.com/kubernetes/charmed-k8s/docs/explain-backups) recommends running juju run etcd/0 etcd.save (or the cis-benchmark action for the control-plane) to create a snapshot, and then copying the file off-cluster.  this etcd snapshot can later be restored if needed.  persistent volumes (pvs) should be backed up using storage-specific methods (ceph rbd snapshots, cloud volume snapshots, or kubernetes backup tools like velero).  while kubernetes can dynamically provision volumes, the underlying data belongs to applications.  in short, you plan backup/restore like any kubernetes system: etcd + pv data.  charmed k8s does not currently include a built-in velero charm, but such tools can be deployed on the cluster via helm or operators.

## networking and storage integrations

network: charmed k8s uses a container networking interface (cni) plugin for pod networking.  by default it deploys calico, which supports bgp/overlay networking and network policy.  calico is installed as a subordinate charm to the worker/control-plane and configures the cluster network.  alternately, flannel or canal (flannel+calico) are available charms.  to switch, one uses a juju overlay that removes the default calico relation and adds the new cni charm.  for l2 load balancing (for services of type=loadbalancer), metallb can be deployed (canonical provides metallb-speaker and metallb-controller charms) and ip addresses allocated for vips on bare-metal.  in the cloud, operators typically use the cloud’s own lb (via the aws/azure/gcp integrator charms).  for ingress to cluster services, any ingress controller (nginx, contour, traefik, etc.) can be deployed via its juju charm or a helm chart.

storage: charmed k8s integrates with block and file storage via csi drivers.  for example, to use ceph, an operator deploys ceph-mon and ceph-osd charms to form a ceph cluster, then the ceph-csi charm to provide rbd and cephfs volume classes.  the bundles in charmhub often include these for ease.  pvs are then provisioned by creating persistentvolumeclaims using those storage classes.  other csi drivers (aws ebs, azure disk, csi-csi for gcp) can also be used via cloud provider operator charms.  juju itself can also manage storage integrations: for instance, the csi:rbd storage relation from kubernetes-control-plane to ceph-rbd populates the necessary secrets and endpoints in the cluster.  in practice, any kubernetes volume type is supported as long as there’s a charms or cloud service backing it.
```
networking:
-----------
[kubernetes-worker] <---> [cni plugin: calico/flannel/cilium]
                              |
                              v
                      [pod to pod networking]
                              |
                              v
                  [metallb or cloud lb or ingress or something]

storage:
--------
[kubernetes-control-plane / worker]
            |
            v
      [ceph-csi charm]
            |
            v
  [ceph cluster: ceph-mon + ceph-osd]

```

## observability stack

charmed k8s supports a full monitoring and logging stack, typically deployed as additional charms in the model.  for metrics, the recommended approach is prometheus + grafana with telegraf.  one can deploy the prometheus2 charm and grafana charm, then relate them: for example, prometheus:grafana-source to grafana:grafana-source for dashboards.  the workers and control-plane expose prometheus endpoints via the scrape interface, so adding relations such as [kubernetes-worker:scrape, prometheus:scrape] and [kubernetes-control-plane:prometheus, prometheus:manual-jobs] wires in all metrics.  telegraf can be related to collect node-level metrics (juju-info) from each unit.  the official charmed k8s monitoring overlay (shown below) demonstrates these relations.  once set up, grafana provides cluster and node dashboards (built-in in the charm, or via grafana labs’ kubernetes dashboards).

for logging, charmed k8s originally recommended graylog (elk-like) via filebeat forwarding.  the docs include a graylog bundle overlay with graylog, elasticsearch, mongodb, and filebeat charms.  however, canonical now favors loki (the “prometheus for logs”) as part of the new lma2 stack.  loki (and grafana agent) can be deployed and related to gather logs: for example, each juju unit forwards its logs to loki.  alerting is handled by alertmanager (charmed alertmanager-k8s), which is integrated with prometheus (alerts firing based on prom rules) and loki.  grafana is the single pane for both metrics and logs.

&#x20;the figure above is an example observability model diagram: a lightweight k8s cluster running the canonical observability stack (prometheus, loki, grafana, grafana agent, alertmanager).  the grafana agent collects telemetry from all components.  loki ingests logs, prometheus ingests metrics, and grafana unifies visualization.  this juju-based lma (logs/metrics/alerts) architecture makes monitoring of charmed k8s clusters scalable and self-hosted.

in summary, charmed k8s provides integrated observability: built-in support for prometheus endpoints on each service, easy deployment of prometheus/grafana loki charms, and even grafana dashboards out-of-the-box.  alerts are defined in the alertmanager charm and can be fine-tuned as needed.  telegraf or grafana agent handle system metrics, while loki (or graylog) collects container/app logs.

## add-ons and ecosystem

beyond core kubernetes, charmed k8s integrates with a wide ecosystem:

* microk8s: although a separate distribution, microk8s is canonical’s lightweight single-node kubernetes (installed via snap).  it is often used for edge/iot or development.  canonical provides charms and tools to manage fleets of microk8s as well.  in some workflows, charmed k8s can interact with microk8s clusters (via azure arc or multi-model juju setups for example).
* charmed operators for apps: the juju charmhub contains many “charmed operators” for cloud-native apps (databases, messaging, ml platforms, etc.).  these can be deployed on top of charmed k8s or related into it.  for example, canonical offers charmed kubeflow and charmed kafka, which use juju to deploy complex app stacks on the cluster.
* service mesh (istio): canonical provides an istio bundle.  the istio charm (or sub-charms like istio-gateway, istio-pilot) can be deployed on a charmed k8s cluster to enable service mesh.  for instance, juju deploy istio (bundle) installs istio components which then mesh the services.  (see charmhub “istio bundle” for details.)  once related (like labeling namespaces), istio functionality (sidecar injection, ingress gateway) is enabled.
* serverless (knative): canonical maintains knative charms: knative-operator, knative-serving, knative-eventing.  operators can deploy istio first (as knative requires a mesh), then deploy these knative charms via juju.  this sets up kubernetes-native serverless on charmed k8s, enabling auto-scaling to zero, eventing, etc.  metrics for knative are scraped by prometheus (via relations shown in the knative operators doc).
* edge/iot use cases: charmed k8s supports arm architectures and low-touch deploys, making it suitable for edge.  the distribution is “fully-conformant, low-touch, self-healing kubernetes at the edge”.  juju can deploy to arm servers (or lxd containers on arm), and charms like metallb work on multi-arch.  the same model-driven ease is valuable for telco edge, retail kiosks, etc.  canonical has documented reference architectures for telco and multi-access edge, often using charmed k8s atop maas or public cloud.

## security model

security is integrated at multiple levels in charmed k8s:

* rbac: by default, charmed kubernetes enables rbac for api access.  the kubernetes-control-plane charm sets the api server’s --authorization-mode=node,rbac.  the “alwaysallow” mode is disabled, and a webhook token authenticator (via --token-auth-file disabled and migrated to secrets) ensures only proper credentials are used.  node and pod restrictions are also enabled (like noderestriction admission plugin).  the k8s dashboard and other add-ons should be configured with least-privilege roles.  (juju’s own permissions control access to the cluster via the controller.)
* tls: all intra-cluster communication is secured by tls.  the easyrsa charm acts as a certificate authority, issuing certificates to all kubernetes components.  the kubernetes api, etcd cluster, kubelets, and controller components all use tls for mutual authentication.  for example, etcd is “tls-terminated” in its charm, meaning client and peer traffic is encrypted.  if desired, vault can replace easyrsa for certificate management by deploying a vault charm and relating it.
* secrets management: for storing sensitive data, you can deploy a vault (hashicorp) charm, which provides a secure secret store for the cluster.  the vault charm is listed as compatible with charmed k8s.  additionally, charmed k8s can integrate opa gatekeeper or the new opa policy controller to enforce policies on secrets, configs, etc.  (recent releases include an opa policy-controller charm as noted in release notes.)
* compliance and hardening: charmed k8s adheres to cis kubernetes benchmark by default.  the charms have been audited: for example, the control-plane charm’s cis profile ensures --profiling is disabled and a kubernetes encryption config is enabled.  the worker charm enables kernel hardening flags (protect-kernel-defaults) and disables the insecure kubelet port.  the built-in cis-benchmark juju action can be run on each unit to report compliance, and auto-remediation flags are available.  because charmed k8s uses ubuntu, common security features like apparmor profiles, seccomp, and esm are inherited.  canonical also provides an “ubuntu cis-compliant kubernetes” documentation set for operators to lock down their cluster.

## ci/cd integration and automation

charmed k8s fits into modern ci/cd workflows through gitops and operators.  for gitops, juju can be integrated with systems like azure arc or flux: for example, canonical announced azure arc support, where charmed k8s clusters can be managed declaratively via git repos.  in practice, this means cluster manifests (juju bundles, charms channels, kubernetes yaml) can be kept in git, and changes applied via automation.  juju itself can be scripted: one could run juju cli commands in github actions or jenkins pipelines to spin up a model, deploy charms, or relate services.  for instance, operators often use juju’s github actions (see juju’s examples) to perform repeatable deploys.

for pipeline automation, jenkins can be deployed on the kubernetes cluster using the jenkins-k8s-operator charm.  this provides a continuous integration server managed by juju; pipeline jobs can be created to build and test applications on the cluster.  the jenkins kubernetes operator allows dynamic provisioning of jenkins agents as pods.  alternatively, teams often run jenkins or other ci servers outside the cluster and use juju or kubectl calls in their pipeline scripts.  canonical’s own ci for charmed k8s uses jenkins and github actions (like the charmed-kubernetes/jenkins repository) to build and test releases.

in short, charmed k8s does not mandate a specific ci tool – it is fully compatible with gitops (via flux/argo/github actions for kubernetes apps, and juju-managed gitops for infra) and with jenkins pipelines.  the jenkins-k8s-operator and community github action examples show how tightly juju can integrate with ci/cd pipelines.
```
[ git repo ]
     |
     v
[ gitHub actions / jenkins ]
     |
     v
[ juju cli scripts / flux / argo ]
     |
     v
[ juju controller ]
     |
     v
[ charmed k8s deployment ]

```

## troubleshooting and best practices

monitoring health: in day-to-day ops, use juju tools to inspect the cluster.  juju status --color gives an overview: each charm’s workload and agent status, units and machines.  a healthy deployment will show all units as active (green) with messages like “ready” or “kubernetes master running”.  if something is down, juju status will show error or blocked.  you can narrow focus to a component.  the juju agent logs and machine logs can be viewed via juju debug-log, which tails logs across all units.  useful filters (by unit or level) help pinpoint errors.

inspecting logs: by default, juju centralizes unit logs on the controller, but you can also juju ssh into any unit’s machine and inspect logs under /var/log/juju/.  for kubernetes-level logs (pod logs), use kubectl logs as usual.  for cluster logs (audit, system components), you may use the graylog or loki stack mentioned above.  if a control-plane or etcd issue arises, check those unit’s logs directly with juju ssh kubernetes-control-plane/0 and journalctl -u snap.kube-apiserver.

useful tools: the juju community provides juju-crashdump, a script that collects a complete bundle of logs, configs, and status from a model.  running juju-crashdump (with debug-layer and config flags) on your controller creates a tarball that is very helpful for diagnosing complex failures or when filing bug reports.  this captures snap versions, systemd output, unit counters, etc.

common issues: a few caveats have been observed in production: some deployments on the localhost (lxd) cloud sometimes suffer lxd profile mismatches on charm upgrades.  if units in lxd fail to restart after an upgrade, check that the auto-generated lxc profiles (shown via juju machines --format=yaml) have the correct nesting and mount settings, and manually fix with lxc profile edit if needed.  another known issue is helm v2 (tiller) hitting a lb bug on juju; the troubleshooting docs cover how to redeploy helm v2 or skip tiller in favor of helm v3.

best practices for working with charmed k8s:

* always use juju relations for any integration (do not, for example, manually configure access outside of juju).
* keep juju and models up-to-date (juju itself can be upgraded via snap refresh juju).
* use snap refresh schedules to control when os and kubernetes updates apply.
* regularly run the cis-benchmark actions to ensure no drift from secure settings.
* test upgrades in a staging model first.

in field deployments, operators find charmed k8s reliable, failures are rare because juju restarts crashed services, relocates units, or triggers leader re-election.  most day-to-day troubleshooting is just examining juju’s status and logs, applying charm actions and letting charms handle the details.
