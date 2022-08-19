# **Hosted Clusters on Red Hat Advanced Cluster Management**

<img src="hypershift.jpg" style="width: 1000px;" border=0/>

 Red Hat Advanced Cluster Management for Kubernetes version 2.6 with the Multicluster Engine Operator 2.1 can deploy Red Hat OpenShift Container Platform clusters by using two different control plane configurations. The standalone configuration uses multiple dedicated virtual machines or physical machines to host the OpenShift Container Platform control plane.  One can also deploy hosted control planes to provision the OpenShift Container Platform control plane as pods on a hosting service cluster without the need for dedicated physical machines for each control-plane.

Note: This feature also works with the Multicluster Engine Operator 2.1 without Red Hat Advanced Cluster Management for Kubernetes.  However additional configuration of the Infrastructure Operator is required.

For Red Hat Advanced Cluster Management, Amazon Web Services and bare metal is supported as technology preview. One can host the control planes for your Red Hat OpenShift Container Platform version 4.10.7 and later.

The control plane is run as pods that are contained in a single namespace and is associated with the hosted control plane cluster. When OpenShift Container Platform provisions this type of hosted cluster, it provisions a worker node independent of the control plane.

The following benefits are yielded when using hosted control plane clusters:

 * Lowers cost by eliminating the need to host dedicated control plane nodes
 * Enables separation between the control plane and the workloads for improved isolation
 * Significantly reduces cluster provision time by removing control-plane node bootstrapping
 * Supports vanilla deployments or fully customized OpenShift provisioning    





Before we walk through a deployment of a hosted cluster lets first take a quick look at the lab environment.   The hub cluster where RHACM 2.6 is running is an OpenShift 4.10.26 compact 3 node cluster.  On this cluster we are runing the following

~~~bash
$ oc get infraenv -n kni21
NAME    ISO CREATED AT
kni21   2022-08-16T20:50:20Z
~~~

~~~bash
$ oc get agents -n kni21
NAME                                   CLUSTER   APPROVED   ROLE          STAGE
07f25812-6c5b-ece8-a4d5-9a3c2c76fa3a             false      auto-assign   
304d2046-5a32-4cec-8c08-96a2b76a6537             true       auto-assign   
4ca57efe-0940-4fd6-afcb-7db41b8c918b             true       auto-assign   
65f41daa-7ea8-4637-a7c6-f2cde634404a             true       auto-assign   
cef2bbd6-f974-5ecf-331e-db11391fd7a5             false      auto-assign   
d1e0c4b8-6f70-8d5b-93a3-706754ee2ee9             false      auto-assign   
~~~



~~~bash
$ oc patch mce multiclusterengine --type=merge -p '{"spec":{"overrides":{"components":[{"name":"hypershift-preview","enabled": true}]}}}'
multiclusterengine.multicluster.openshift.io/multiclusterengine patched
~~~

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

~~~bash
$ oc apply -f import-hub-kni20.yaml 
Warning: resource managedclusters/local-cluster is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by oc apply. oc apply should only be used on resources created declaratively by either oc create --save-config or oc apply. The missing annotation will be patched automatically.
managedcluster.cluster.open-cluster-management.io/local-cluster configured
~~~

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

~~~bash
$ oc apply -f managed-cluster-addon-kni20.yaml
managedclusteraddon.addon.open-cluster-management.io/hypershift-addon created
~~~

~~~bash
$ oc get managedclusteraddons -n local-cluster hypershift-addon
NAME               AVAILABLE   DEGRADED   PROGRESSING
hypershift-addon   True 
~~~

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

~~~bash
$ oc create -f capi-role-kni21.yaml 
role.rbac.authorization.k8s.io/capi-provider-role created
~~~

~~~bash
$ cat hosted-cluster-kni21.yaml 
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
~~~

~~~bash
$ oc create -f hosted-cluster-kni21.yaml 
hostedcluster.hypershift.openshift.io/kni21 created
secret/pullsecret-cluster-kni21 created
secret/sshkey-cluster-kni21 created
nodepool.hypershift.openshift.io/nodepool-kni21-1 created
managedcluster.cluster.open-cluster-management.io/kni21 created
klusterletaddonconfig.agent.open-cluster-management.io/kni21 created
~~~

~~~bash
$ oc extract -n kni21 secret/kni21-admin-kubeconfig --to=- > kni21-kubeconfig
# kubeconfig
~~~
