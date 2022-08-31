# **Hosted Clusters on Red Hat Advanced Cluster Management**

<img src="hypershift.jpg" style="width: 1000px;" border=0/>

 Red Hat Advanced Cluster Management for Kubernetes version 2.6 with the Multicluster Engine Operator 2.1 can deploy Red Hat OpenShift Container Platform clusters by using two different control plane configurations. The standalone configuration uses multiple dedicated virtual machines or physical machines to host the OpenShift Container Platform control plane.  One can also deploy hosted control planes to provision the OpenShift Container Platform control plane as pods on a hosting service cluster without the need for dedicated physical machines for each control-plane.

Note: This feature also works with the Multicluster Engine Operator 2.1 without Red Hat Advanced Cluster Management for Kubernetes.  However additional configuration of the Infrastructure Operator is required.

For Red Hat Advanced Cluster Management, Amazon Web Services and bare metal is supported as technology preview. One can host the control planes for your Red Hat OpenShift Container Platform version 4.10.7 and later.

The control plane is run as pods that are contained in a single namespace and is associated with the hosted control plane cluster. When OpenShift Container Platform provisions this type of hosted cluster, it provisions a worker node independent of the control plane.

## Benefits

The following benefits are yielded when using hosted control plane clusters:

 * Lowers cost by eliminating the need to host dedicated control plane nodes
 * Enables separation between the control plane and the workloads for improved isolation
 * Significantly reduces cluster provision time by removing control-plane node bootstrapping
 * Supports vanilla deployments or fully customized OpenShift provisioning    

## Lab Environment

Before we walk through a deployment of a hosted cluster lets first take a quick look at the lab environment.   The hub cluster where RHACM 2.6 is running is an OpenShift 4.10.26 compact 3 node bare metal cluster.  On this cluster we are runing the following operators:

 * OpenShift Local Storage
 * Openshift Data Foundation
 * Advanced Cluster Management for Kubernetes
 * Multicluster Enginer for Kubernetes

For convenience we have already gone ahead and configured a infrastructure environment called kni21:

~~~bash
$ oc get infraenv -n kni21
NAME    ISO CREATED AT
kni21   2022-08-16T20:50:20Z
~~~

Inside of the infrastructure environment we have already gone ahead and used the host discovery to bring online 6 bare metal nodes, 3 of which we have marked as approved for availability.   These 3 nodes will become the workers for our hosted cluster we plan to deploy in future steps of this blog.

~~~bash
$ oc get agents -n kni21
NAME                                   CLUSTER   APPROVED   ROLE          STAGE 
304d2046-5a32-4cec-8c08-96a2b76a6537             true       auto-assign   
4ca57efe-0940-4fd6-afcb-7db41b8c918b             true       auto-assign   
65f41daa-7ea8-4637-a7c6-f2cde634404a             true       auto-assign     
~~~

Now that we have an understanding how the lab environment is preconfigured we can move onto enabling the hosted cluster feature.

## Enable Hosted Clusters Feature

Since hosted cluster are a technology preview we need to enable them on the hub cluster with the following steps.  The first step is to patch the multiclusterengine and enable the hypershift component:

~~~bash
$ oc patch mce multiclusterengine --type=merge -p '{"spec":{"overrides":{"components":[{"name":"hypershift-preview","enabled": true}]}}}'
multiclusterengine.multicluster.openshift.io/multiclusterengine patched
~~~

Once we have patched the multiclusterengine we need to next configure a managed cluster custom resource for our hub cluster:

~~~bash
$ cat << EOF > ~/import-hub-kni20.yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  labels:
    local-cluster: "true"
  name: local-cluster
spec:
  hubAcceptsClient: true
  leaseDurationSeconds: 60
EOF
~~~

With the managed cluster file created we can now apply it to the hub cluster:

~~~bash
$ oc apply -f import-hub-kni20.yaml 
Warning: resource managedclusters/local-cluster is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by oc apply. oc apply should only be used on resources created declaratively by either oc create --save-config or oc apply. The missing annotation will be patched automatically.
managedcluster.cluster.open-cluster-management.io/local-cluster configured
~~~

The final configuration step is to enable the managed cluster addon component for hypershift.  Do do this we will create a custom resource like the one below:

~~~bash
$ cat << EOF > ~/managed-cluster-addon-kni20.yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: hypershift-addon
  namespace: local-cluster
spec:
  installNamespace: open-cluster-management-agent-addon
EOF
~~~

Then apply the custom resource file we created above to the hub cluster:

~~~bash
$ oc apply -f managed-cluster-addon-kni20.yaml
managedclusteraddon.addon.open-cluster-management.io/hypershift-addon created
~~~

Finally after a few minutes we should see the manage cluster addon for hypershift listed as available:

~~~bash
$ oc get managedclusteraddons -n local-cluster hypershift-addon
NAME               AVAILABLE   DEGRADED   PROGRESSING
hypershift-addon   True 
~~~

## Deploying Hosted Cluster on Bare Metal

A hosted cluster is a cluster whose control plane is deployed on another OpenShift installation.   Since we just finished configuring the hypershift addon in our hub cluster we can now deploy a hosted cluster.   To do that we first need to create a capi provider role in the namespace where we have our discovered hosts.  In this case its the kni21 namespace which also happens to be the name of our hosted cluster we will be deploying.  The capi provider role custom resource file below provides the ability for us to actually see and consume the discovered hosts.  

~~~bash
cat << EOF > ~/capi-role-kni21.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: capi-provider-role
  namespace: kni21
rules:
- apiGroups:
  - agent-install.openshift.io
  resources:
  - agents
  verbs:
  - '*'
EOF
~~~

Once we have created the capi provider role custom resource file lets go ahead and create it against our hub cluster:

~~~bash
$ oc create -f capi-role-kni21.yaml 
role.rbac.authorization.k8s.io/capi-provider-role created
~~~

With the role in place we can go forward creating our hosted cluster custom resource file for kni21.  This custom resource file will have multiple sections.   Lets break down those sections and their purpose here:

 * The HostedCluster section of the file defines resources like the cluster name, namespace, release image for the cluster control plane, pull-secret location, sshkey location, cluster dns domain, cluster networking and the various publishing strategies.  I should point out in this configuration we are using the Nodeport API publishing strategy and so the ipaddress defined is just one of the ipaddresses off a worker node on the cluster hosting kni21's control plane.
 *  The Secret(s) section of the file defines two secrets: one being the pull-secret the hosted cluster will use and the other being the ssk public key the hosted cluster will use for any deployed worker nodes.
 *  The NodePool section defines the name of the nodepool, the namespace the nodepool should reside in, the number of replicas (workers) to create and the release image for the worker nodes (normally it should be the same image used from the HostedCluster section).
 *  The ManagedCluster section defines the cluster type (hypershift), the namespace and the hub will accept us as a client
 *  The KlusterletAddonConfig section defines what additional management capabilities the hub cluster should have visibility to on this hosted cluster.  The standards are defined there from a Red Hat Advanced Cluster Management perspective.

Now that we have a brief understanding of what the custom resource file does lets go ahead and create it:

~~~bash
$ cat << EOF > ~/hosted-cluster-kni21.yaml 
---
apiVersion: hypershift.openshift.io/v1alpha1
kind: HostedCluster
metadata:
  name: 'kni21'
  namespace: 'kni21'
  labels:
    "cluster.open-cluster-management.io/clusterset": 'default'
spec:
  release:
    image: quay.io/openshift-release-dev/ocp-release:4.10.26-x86_64
  pullSecret:
    name: pullsecret-cluster-kni21
  sshKey:
    name: sshkey-cluster-kni21
  networking:
    podCIDR: 10.132.0.0/14
    serviceCIDR: 172.31.0.0/16
    machineCIDR: 192.168.0.0/24
    networkType: OpenShiftSDN
  platform:
    type: Agent
    agent:
      agentNamespace: 'kni21'
  infraID: 'kni21'
  dns:
    baseDomain: 'schmaustech.com'
  services:
  - service: APIServer
    servicePublishingStrategy:
      type: NodePort
      nodePort:
        address: 192.168.0.210
        port: 30000
  - service: OAuthServer
    servicePublishingStrategy:
      type: Route
  - service: OIDC
    servicePublishingStrategy:
      type: Route
  - service: Konnectivity
    servicePublishingStrategy:
      type: Route
  - service: Ignition
    servicePublishingStrategy:
      type: Route
---
apiVersion: v1
kind: Secret
metadata:
  name: pullsecret-cluster-kni21
  namespace: kni21
data:
  '.dockerconfigjson': eyJhdXRocyI6eyJwcm92aXNpb25pbmcuc2NobWF1c3RlY2guY29tOjUwMDAiOnsiYXV0aCI6IlpIVnRiWGs2WkhWdGJYaz0iLCJlbWFpbCI6ImJzY2htYXVzQHNjaG1hdXN0ZWNoLmNvbSJ9fX0=
type: kubernetes.io/dockerconfigjson
---
apiVersion: v1
kind: Secret
metadata:
  name: sshkey-cluster-kni21
  namespace: 'kni21'
stringData:
  id_rsa.pub: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCoy2/8SC8K+9PDNOqeNady8xck4AgXqQkf0uusYfDJ8IS4pFh178AVkz2sz3GSbU41CMxO6IhyQS4Rga3Ft/VlW6ZAW7icz3mw6IrLRacAAeY1BlfxfupQL/yHjKSZRze9vDjfQ9UDqlHF/II779Kz5yRKYqXsCt+wYESU7DzdPuGgbEKXrwi9GrxuXqbRZOz5994dQW7bHRTwuRmF9KzU7gMtMCah+RskLzE46fc2e4zD1AKaQFaEm4aGbJjQkELfcekrE/VH3i35cBUDacGcUYmUEaco3c/+phkNP4Iblz4AiDcN/TpjlhbU3Mbx8ln6W4aaYIyC4EVMfgvkRVS1xzXcHexs1fox724J07M1nhy+YxvaOYorQLvXMGhcBc9Z2Au2GA5qAr5hr96AHgu3600qeji0nMM/0HoiEVbxNWfkj4kAegbItUEVBAWjjpkncbe5Ph9nF2DsBrrg4TsJIplYQ+lGewzLTm/cZ1DnIMZvTY/Vnimh7qa9aRrpMB0= bschmaus@provisioning
---
apiVersion: hypershift.openshift.io/v1alpha1
kind: NodePool
metadata:
  name: 'nodepool-kni21-1'
  namespace: 'kni21'
spec:
  clusterName: 'kni21'
  replicas: 3
  management:
    autoRepair: false
    upgradeType: InPlace
  platform:
    type: Agent
    agent:
      agentLabelSelector:
        matchLabels: {}
  release:
    image: quay.io/openshift-release-dev/ocp-release:4.10.26-x86_64
---
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  labels:
    cloud: hypershift
    name: 'kni21'
    cluster.open-cluster-management.io/clusterset: 'default'
  name: 'kni21'
spec:
  hubAcceptsClient: true
---
apiVersion: agent.open-cluster-management.io/v1
kind: KlusterletAddonConfig
metadata:
  name: 'kni21'
  namespace: 'kni21'
spec:
  clusterName: 'kni21'
  clusterNamespace: 'kni21'
  clusterLabels:
    cloud: ai-hypershift
  applicationManager:
    enabled: true
  policyController:
    enabled: true
  searchCollector:
    enabled: true
  certPolicyController:
    enabled: true
  iamPolicyController:
    enabled: true
EOF
~~~

With the file created lets go ahead and create it against our hub cluster.  Keep in mind once this step is run it will begin the deployment process of our hosted cluster called kni21:

~~~bash
$ oc create -f hosted-cluster-kni21.yaml 
hostedcluster.hypershift.openshift.io/kni21 created
secret/pullsecret-cluster-kni21 created
secret/sshkey-cluster-kni21 created
nodepool.hypershift.openshift.io/nodepool-kni21-1 created
managedcluster.cluster.open-cluster-management.io/kni21 created
klusterletaddonconfig.agent.open-cluster-management.io/kni21 created
~~~

Now that we have created the kni21 cluster the process should begin to deploy the hosted cluster control plane on our hub cluster.  Along with this the deployment process will also obtain 3 baremetal worker nodes from the kni21 infraenv that were marked as available and incorporate them into our kni21 cluster.  However don't take my word for it lets go ahead and see what is happening as the cluster deploys.

## Observing Deployment Progression

During the hosted cluster deployment process we view a few different commands that show the status and progression of the cluster.  The commands below are just a snapshot in time during the cluster deployment so the output can vary.   These commands can also be helpful in troubleshooting should the deploy not be progressing.  First we can look at the hostedcluster output:

~~~bash
$ oc get hostedcluster -n kni21
NAME    VERSION   KUBECONFIG               PROGRESS   AVAILABLE   REASON                    MESSAGE
kni21             kni21-admin-kubeconfig   Partial    True        HostedClusterAsExpected 
~~~

And then if we want to see a deeper detail on the hostedcluster to see status of the various stages:

~~~bash
$ oc get hostedcluster -n kni21 -o yaml
apiVersion: v1
items:
- apiVersion: hypershift.openshift.io/v1alpha1
  kind: HostedCluster
  metadata:
    creationTimestamp: "2022-08-18T19:12:25Z"
    finalizers:
    - hypershift.openshift.io/finalizer
    generation: 3
    labels:
      cluster.open-cluster-management.io/clusterset: default
    name: kni21
    namespace: kni21
    resourceVersion: "163368743"
    uid: 18f08f34-181d-4c0c-b110-6128e7fd9cc5
  spec:
    autoscaling: {}
    clusterID: 9fea8391-75f4-46f8-80fa-a59d8eef3832
    controllerAvailabilityPolicy: SingleReplica
    dns:
      baseDomain: schmaustech.com
    etcd:
      managementType: Managed
    fips: false
    infraID: kni21
    infrastructureAvailabilityPolicy: SingleReplica
    issuerURL: https://kubernetes.default.svc
    networking:
      machineCIDR: 192.168.0.0/24
      networkType: OpenShiftSDN
      podCIDR: 10.132.0.0/14
      serviceCIDR: 172.31.0.0/16
    olmCatalogPlacement: management
    platform:
      agent:
        agentNamespace: kni21
      type: Agent
    pullSecret:
      name: pullsecret-cluster-kni21
    release:
      image: quay.io/openshift-release-dev/ocp-release:4.10.26-x86_64
    services:
    - service: APIServer
      servicePublishingStrategy:
        nodePort:
          address: 192.168.0.210
          port: 30000
        type: NodePort
    - service: OAuthServer
      servicePublishingStrategy:
        type: Route
    - service: OIDC
      servicePublishingStrategy:
        type: Route
    - service: Konnectivity
      servicePublishingStrategy:
        type: Route
    - service: Ignition
      servicePublishingStrategy:
        type: Route
    sshKey:
      name: sshkey-cluster-kni21
  status:
    conditions:
    - lastTransitionTime: "2022-08-18T19:15:03Z"
      message: ""
      observedGeneration: 3
      reason: AsExpected
      status: "True"
      type: ClusterVersionSucceeding
    - lastTransitionTime: "2022-08-18T19:12:25Z"
      message: ""
      observedGeneration: 3
      reason: ClusterVersionStatusUnknown
      status: Unknown
      type: ClusterVersionUpgradeable
    - lastTransitionTime: "2022-08-18T19:14:20Z"
      message: ""
      observedGeneration: 3
      reason: HostedClusterAsExpected
      status: "True"
      type: Available
    - lastTransitionTime: "2022-08-18T19:12:25Z"
      message: Configuration passes validation
      observedGeneration: 3
      reason: HostedClusterAsExpected
      status: "True"
      type: ValidConfiguration
    - lastTransitionTime: "2022-08-18T19:12:25Z"
      message: HostedCluster is support by operator configuration
      observedGeneration: 3
      reason: HostedClusterAsExpected
      status: "True"
      type: SupportedHostedCluster
    - lastTransitionTime: "2022-08-18T19:13:05Z"
      message: Configuration passes validation
      reason: HostedClusterAsExpected
      status: "True"
      type: ValidHostedControlPlaneConfiguration
    - lastTransitionTime: "2022-08-18T19:13:39Z"
      message: ""
      observedGeneration: 3
      reason: IgnitionServerDeploymentAsExpected
      status: "True"
      type: IgnitionEndpointAvailable
    - lastTransitionTime: "2022-08-18T19:12:25Z"
      message: Reconciliation active on resource
      observedGeneration: 3
      reason: ReconciliationActive
      status: "True"
      type: ReconciliationActive
    - lastTransitionTime: "2022-08-18T19:12:28Z"
      message: Release image is valid
      observedGeneration: 3
      reason: AsExpected
      status: "True"
      type: ValidReleaseImage
    - lastTransitionTime: "2022-08-18T19:12:25Z"
      message: ""
      observedGeneration: 3
      reason: ReconciliatonSucceeded
      status: "True"
      type: ReconciliationSucceeded
    ignitionEndpoint: ignition-server-kni21-kni21.apps.kni20.schmaustech.com
    kubeadminPassword:
      name: kni21-kubeadmin-password
    kubeconfig:
      name: kni21-admin-kubeconfig
    version:
      desired:
        image: quay.io/openshift-release-dev/ocp-release:4.10.26-x86_64
      history:
      - completionTime: null
        image: quay.io/openshift-release-dev/ocp-release:4.10.26-x86_64
        startedTime: "2022-08-18T19:12:25Z"
        state: Partial
        verified: false
        version: ""
      observedGeneration: 1
kind: List
metadata:
  resourceVersion: ""
~~~

We can also look at the agents to see if they have been assigned to the cluster yet.  In the example below run before they were automatically assigned we can see the cluster column is still absent of the kni21 cluster name:

~~~bash
$ oc get agents -n kni21
NAMESPACE   NAME                                   CLUSTER   APPROVED   ROLE          STAGE
kni21       304d2046-5a32-4cec-8c08-96a2b76a6537             true       auto-assign   
kni21       4ca57efe-0940-4fd6-afcb-7db41b8c918b             true       auto-assign   
kni21       65f41daa-7ea8-4637-a7c6-f2cde634404a             true       auto-assign  
~~~

We can also see that nodepool status:

~~~bash
$ oc get nodepool -A
NAMESPACE   NAME               CLUSTER   DESIRED NODES   CURRENT NODES   AUTOSCALING   AUTOREPAIR   VERSION   UPDATINGVERSION   UPDATINGCONFIG   MESSAGE
kni21       nodepool-kni21-1   kni21     3               3               False 
~~~

## Validating Cluster Installation

Once deployment has completed we can validate the cluster.   First need to obtain the kubeconfig for our new cluster.  To do this we can run the following extract command a few minutes after we created our hosted cluster:

~~~bash
$ oc extract -n kni21 secret/kni21-admin-kubeconfig --to=- > kubeconfig-kni21
# kubeconfig
~~~

Now that we have the kubeconfig we can use it to view the cluster operators of our hostedcluster:

~~~bash
$ oc get co --kubeconfig=kubeconfig-kni21
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
console                                    4.10.26   True        False         False      2m38s   
csi-snapshot-controller                    4.10.26   True        False         False      4m3s    
dns                                        4.10.26   True        False         False      2m52s   
image-registry                             4.10.26   True        False         False      2m8s    
ingress                                    4.10.26   True        False         False      22m     
kube-apiserver                             4.10.26   True        False         False      23m     
kube-controller-manager                    4.10.26   True        False         False      23m     
kube-scheduler                             4.10.26   True        False         False      23m     
kube-storage-version-migrator              4.10.26   True        False         False      4m52s   
monitoring                                 4.10.26   True        False         False      69s     
network                                    4.10.26   True        False         False      4m3s    
node-tuning                                4.10.26   True        False         False      2m22s   
openshift-apiserver                        4.10.26   True        False         False      23m     
openshift-controller-manager               4.10.26   True        False         False      23m     
openshift-samples                          4.10.26   True        False         False      2m15s   
operator-lifecycle-manager                 4.10.26   True        False         False      22m     
operator-lifecycle-manager-catalog         4.10.26   True        False         False      23m     
operator-lifecycle-manager-packageserver   4.10.26   True        False         False      23m     
service-ca                                 4.10.26   True        False         False      4m41s   
storage                                    4.10.26   True        False         False      4m43s 
~~~

Further we can also look at the running pods on our kni21 cluster:

~~~bash
$ oc get pods -A --kubeconfig=kubeconfig-kni21
NAMESPACE                                          NAME                                                      READY   STATUS             RESTARTS        AGE
kube-system                                        konnectivity-agent-khlqv                                  0/1     Running            0               3m52s
kube-system                                        konnectivity-agent-nrbvw                                  0/1     Running            0               4m24s
kube-system                                        konnectivity-agent-s5p7g                                  0/1     Running            0               4m14s
kube-system                                        kube-apiserver-proxy-asus3-vm1.kni.schmaustech.com        1/1     Running            0               5m56s
kube-system                                        kube-apiserver-proxy-asus3-vm2.kni.schmaustech.com        1/1     Running            0               6m37s
kube-system                                        kube-apiserver-proxy-asus3-vm3.kni.schmaustech.com        1/1     Running            0               6m17s
openshift-cluster-node-tuning-operator             cluster-node-tuning-operator-798fcd89dc-9cf2k             1/1     Running            0               20m
openshift-cluster-node-tuning-operator             tuned-dhw5p                                               1/1     Running            0               109s
openshift-cluster-node-tuning-operator             tuned-dlp8f                                               1/1     Running            0               110s
openshift-cluster-node-tuning-operator             tuned-l569k                                               1/1     Running            0               109s
openshift-cluster-samples-operator                 cluster-samples-operator-6b5bcb9dff-kpnbc                 2/2     Running            0               20m
openshift-cluster-storage-operator                 cluster-storage-operator-5f784969f5-vwzgz                 1/1     Running            1 (113s ago)    20m
openshift-cluster-storage-operator                 csi-snapshot-controller-6b7687b7d9-7nrfw                  1/1     Running            0               3m8s
openshift-cluster-storage-operator                 csi-snapshot-controller-6b7687b7d9-csksg                  1/1     Running            0               3m9s
openshift-cluster-storage-operator                 csi-snapshot-controller-operator-7f4d9fc5b8-hkvrk         1/1     Running            0               20m
openshift-cluster-storage-operator                 csi-snapshot-webhook-6759b5dc8b-7qltn                     1/1     Running            0               3m12s
openshift-cluster-storage-operator                 csi-snapshot-webhook-6759b5dc8b-f8bqk                     1/1     Running            0               3m12s
openshift-console-operator                         console-operator-8675b58c4c-flc5p                         1/1     Running            1 (96s ago)     20m
openshift-console                                  console-5cbf6c7969-6gk6z                                  1/1     Running            0               119s
openshift-console                                  downloads-7bcd756565-6wj5j                                1/1     Running            0               4m3s
openshift-dns-operator                             dns-operator-77d755cd8c-xjfbn                             2/2     Running            0               21m
openshift-dns                                      dns-default-jwjkz                                         2/2     Running            0               113s
openshift-dns                                      dns-default-kfqnh                                         2/2     Running            0               113s
openshift-dns                                      dns-default-xlqsm                                         2/2     Running            0               113s
openshift-dns                                      node-resolver-jzxnd                                       1/1     Running            0               110s
openshift-dns                                      node-resolver-xqdr5                                       1/1     Running            0               110s
openshift-dns                                      node-resolver-zl6h4                                       1/1     Running            0               110s
openshift-image-registry                           cluster-image-registry-operator-64fcfdbf5-r7d5t           1/1     Running            0               20m
openshift-image-registry                           image-registry-7fdfd99d68-t9pq9                           1/1     Running            0               53s
openshift-image-registry                           node-ca-hkfnr                                             1/1     Running            0               56s
openshift-image-registry                           node-ca-vlsdl                                             1/1     Running            0               56s
openshift-image-registry                           node-ca-xqnsw                                             1/1     Running            0               56s
openshift-ingress-canary                           ingress-canary-86z6r                                      1/1     Running            0               4m13s
openshift-ingress-canary                           ingress-canary-8jhxk                                      1/1     Running            0               3m52s
openshift-ingress-canary                           ingress-canary-cv45h                                      1/1     Running            0               4m24s
openshift-ingress                                  router-default-6bb8944f66-z2lxr                           1/1     Running            0               20m
openshift-kube-storage-version-migrator-operator   kube-storage-version-migrator-operator-56b57b4844-p9zgp   1/1     Running            1 (2m16s ago)   20m
openshift-kube-storage-version-migrator            migrator-58bb4d89d5-5sl9w                                 1/1     Running            0               3m30s
openshift-monitoring                               alertmanager-main-0                                       6/6     Running            0               100s
openshift-monitoring                               cluster-monitoring-operator-5bc5885cd4-dwbc4              2/2     Running            0               20m
openshift-monitoring                               grafana-78f798868c-wd84p                                  3/3     Running            0               94s
openshift-monitoring                               kube-state-metrics-58b8f97f6c-6kp4v                       3/3     Running            0               104s
openshift-monitoring                               node-exporter-ll7cp                                       2/2     Running            0               103s
openshift-monitoring                               node-exporter-tgsqg                                       2/2     Running            0               103s
openshift-monitoring                               node-exporter-z99gr                                       2/2     Running            0               103s
openshift-monitoring                               openshift-state-metrics-677b9fb74f-qqp6g                  3/3     Running            0               104s
openshift-monitoring                               prometheus-adapter-f69fff5f9-7tdn9                        0/1     Running            0               17s
openshift-monitoring                               prometheus-k8s-0                                          6/6     Running            0               93s
openshift-monitoring                               prometheus-operator-6b9d4fd9bd-tqfcx                      2/2     Running            0               2m2s
openshift-monitoring                               telemeter-client-74d599658c-wqw5j                         3/3     Running            0               101s
openshift-monitoring                               thanos-querier-64c8757854-z4lll                           6/6     Running            0               98s
openshift-multus                                   multus-additional-cni-plugins-cqst9                       1/1     Running            0               6m14s
openshift-multus                                   multus-additional-cni-plugins-dbmkj                       1/1     Running            0               5m56s
openshift-multus                                   multus-additional-cni-plugins-kcwl9                       1/1     Running            0               6m14s
openshift-multus                                   multus-admission-controller-22cmb                         2/2     Running            0               3m52s
openshift-multus                                   multus-admission-controller-256tn                         2/2     Running            0               4m13s
openshift-multus                                   multus-admission-controller-mz9jm                         2/2     Running            0               4m24s
openshift-multus                                   multus-bxgvr                                              1/1     Running            0               6m14s
openshift-multus                                   multus-dmkdc                                              1/1     Running            0               6m14s
openshift-multus                                   multus-gqw2f                                              1/1     Running            0               5m56s
openshift-multus                                   network-metrics-daemon-6cx4x                              2/2     Running            0               5m56s
openshift-multus                                   network-metrics-daemon-gz4jp                              2/2     Running            0               6m13s
openshift-multus                                   network-metrics-daemon-jq9j4                              2/2     Running            0               6m13s
openshift-network-diagnostics                      network-check-source-8497dc8f86-cn4nm                     1/1     Running            0               5m59s
openshift-network-diagnostics                      network-check-target-d8db9                                1/1     Running            0               5m58s
openshift-network-diagnostics                      network-check-target-jdbv8                                1/1     Running            0               5m58s
openshift-network-diagnostics                      network-check-target-zzmdv                                1/1     Running            0               5m55s
openshift-network-operator                         network-operator-f5b48cd67-x5dcz                          1/1     Running            0               21m
openshift-sdn                                      sdn-452r2                                                 2/2     Running            0               5m56s
openshift-sdn                                      sdn-68g69                                                 2/2     Running            0               6m
openshift-sdn                                      sdn-controller-4v5mv                                      2/2     Running            0               5m56s
openshift-sdn                                      sdn-controller-crscc                                      2/2     Running            0               6m1s
openshift-sdn                                      sdn-controller-fxtn9                                      2/2     Running            0               6m1s
openshift-sdn                                      sdn-n5jm5                                                 2/2     Running            0               6m
openshift-service-ca-operator                      service-ca-operator-5bf7f9d958-vnqcg                      1/1     Running            1 (2m ago)      20m
openshift-service-ca                               service-ca-6c54d7944b-v5mrw                               1/1     Running            0               3m8s
~~~

With our hosted cluster 
