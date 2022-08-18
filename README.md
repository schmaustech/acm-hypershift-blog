# **Hosted Clusters on Red Hat Advanced Cluster Management**

<img src="hypershift.jpg" style="width: 1000px;" border=0/>

 Red Hat Advanced Cluster Management for Kubernetes version 2.6 with the Multicluster Engine Operator 2.1 can deploy Red Hat OpenShift Container Platform clusters by using two different control plane configurations. The standalone configuration uses multiple dedicated virtual machines or physical machines to host the OpenShift Container Platform control plane.  One can also sion hosted control planes to provision the OpenShift Container Platform control plane as pods on a hosting service cluster without the need for dedicated physical machines for each control-plane.

Note: This feature also works with the multicluster engine operator 2.0 without Red Hat Advanced Cluster Management for Kubernetes.

For Red Hat Advanced Cluster Management, Amazon Web Services is supported as a Technology Preview. You can host the control planes for your Red Hat OpenShift Container Platform version 4.10.7 and later.

The control plane is run as pods that are contained in a single namespace and is associated with the hosted control plane cluster. When OpenShift Container Platform provisions this type of hosted cluster, it provisions a worker node independent of the control plane.

See the following benefits of hosted control plane clusters:

    Saves cost by removing the need to host dedicated control plane nodes
    Introduces separation between the control plane and the workloads, which improves isolation and reduces configuration errors that can require changes
    Significantly decreases the cluster provisioning time by removing the requirement for control-plane node bootstrapping
    Supports turn-key deployments or fully customized OpenShift Container Platform provisioning    





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

~~~
