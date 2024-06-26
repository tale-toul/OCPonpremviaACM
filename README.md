# ACM MIRROR REGISTRY AND ON PREM CLUSTERS

## TABLE OF CONTENTS

* [Introduction](#introduction)
* [Deploying OCP using vSpere IPI on a Connected Environment ](#deploying-ocp-using-vspere-ipi-on-a-connected-environment)
* [Infra Nodes During Cluster Deployment](#infra-nodes-during-cluster-deployment)
  * [Adding An Infra Machine Set](#adding-an-infra-machine-set)
    * [Assigning Infra Workloads to Infra Nodes](#assigning-infra-workloads-to-infra-nodes)
  * [Replacing The Worker Machine Set](#replacing-the-worker-machine-set)
* [Installing RHACM on Infra Nodes](#installing-rhacm-on-infra-nodes)
* [Disconnected Environments](#disconnected-environments)
  * [Setting Up the Environment](#setting-up-the-environment)
  * [Installing the Mirror Registry](#installing-the-mirror-registry)
  * [Uninstalling the Mirror Registry](#uninstalling-the-mirror-registry)
  * [Mirroring Images](#mirroring-images)
  * [Mirroring Operators](#mirroring-operators)
  * [Updating the Images](#updating-the-images)

## Introduction

This documents is specific for deploying components in the [demo.redhat.com](https://demo.redhat.com) platform.

This project provides instructions on how to install a Management cluster using vsphere IPI method, in both a connected and disconnected environments.

There is a section on the creation and use of Infra nodes.

The installation and configuration of the mirror registry is explained for the case of the disconnected environment installation. 

The de ployment of managed OCP clusters through ACM "on premisses" has its own section

## Deploying OCP using vSpere IPI on a Connected Environment 

Create a **VMware Cloud Public Cloud Open Environment** in [demo.redhat.com](https://demo.redhat.com). 

![VMWare Cloud Public Open Env Provisioning](images/image14.png)

The environment will be used to deploy an IPI OCP 4.14 cluster hub cluster.

Make sure to enable DNS records creation for OCP. when ordering the environment

![Enable DNS records for OCP](images/image23.png)

It takes about 20 minutes for the environment to be provisioned and ready.
When the environment is ready, an email is received with information on how to access and use it.

The bastion host automatically created in the environment may not have enough resources to run the clients needed to install Openshift, if that is the case shutdown the bastion and add:

* One additional vCPU, 2 in total
* Increase the memory to 8GB
* Add a disk of 250GB.

Ssh into the bastion node and verify the DNS records for the cluster.  The IPs associated with the API and wildcard \*.apps are public, but they are NATed to internal private IPs in the environment, this is specified in the email received from RHDP:

```
$ dig +short api.glnm2.dynamic.opentlc.com
3.223.59.140
$ dig +short *.apps.glnm2.dynamic.opentlc.com
34.198.235.139
```
Get the installer, oc client and pull secret from [the Red Hat Console](https://console.redhat.com/openshift/install) and copy them to the bastion host. You can download the files directly into the bastion host using curl:
```
curl -LO https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.20/openshift-client-linux-4.14.20.tar.gz

curl -LO https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.20/openshift-install-linux.tar.gz
```

Uncompress the tar files and put them in the running path:

```
$ scp ~/Descargas/openshift-* ~/Descargas/pull-secret.txt   \
  lab-user@bastion-glnm2.glnm2.dynamic.opentlc.com

$ tar xvf openshift-client-linux.tar.gz
README.md
oc
kubectl

$ tar xvf openshift-install-linux.tar.gz
README.md
openshift-install

$ sudo cp -vi openshift-install oc /usr/local/bin
'openshift-install' -> '/usr/local/bin/openshift-install'
'oc' -> '/usr/local/bin/oc'
```

Create an ssh key pair.

```
$ ssh localhost
The authenticity of host 'localhost (::1)' can't be established.
ECDSA key fingerprint is SHA256:AIOgCA9BvbsRJN9NSt0jqJ6xEd4pjlcqyHPwF9aLr3Q.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'localhost' (ECDSA) to the list of known hosts.
lab-user@localhost's password:

$ ssh-keygen -t ed25519 -N '' -f ~/.ssh/ocp
Generating public/private ed25519 key pair.
Your identification has been saved in /home/lab-user/.ssh/ocp.
Your public key has been saved in /home/lab-user/.ssh/ocp.pub.
The key fingerprint is:
...
```

Download the vCenter’s root CA certificates.  Go to the vCenter's base URL, for example **https://portal.vc.opentlc.com/** Click on the link **Download trusted root CA certificates** on the right side of the web page

![vCenter root CA certificates](images/image18.png)

![vCenter root CA certificates 2](images/image38.png)

Copy the downloaded file to the bastion host and extract the compressed file:

```
$ scp download.zip lab-user@bastion-6lzvs.6lzvs.dynamic.opentlc.com:
lab-user@bastion-6lzvs.6lzvs.dynamic.opentlc.com's password: 
download.zip                                                                                                                                                100%   29KB 268.9KB/s   00:00 

bastion$ unzip download.zip
Archive:  download.zip
  inflating: certs/lin/7255df92.0    
  inflating: certs/mac/7255df92.0    
  inflating: certs/win/7255df92.0.crt  
  inflating: certs/lin/75c43eb5.r0   
  inflating: certs/mac/75c43eb5.r0   
  inflating: certs/win/75c43eb5.r0.crl
...
```

Update the bastion's system trust so the vCenter root CA certificates are recognized as valid:

```
$ sudo cp -vi certs/lin/* /etc/pki/ca-trust/source/anchors
'certs/lin/02265526.0' -> '/etc/pki/ca-trust/source/anchors/02265526.0'
'certs/lin/2835d715.0' -> '/etc/pki/ca-trust/source/anchors/2835d715.0'
...

$ sudo update-ca-trust extract
```

Create the install-config.yaml file. 

This repository contains a reference install-config.yaml file for a connected vSphere IPI installation at **IPI/Orig-install-config.yaml**

To create the initial configuration file use the following command.  The information required is in the email from RHDP.  For the VIP for API and Ingress use the NAT IP described in the email. The cluster name is the GUID described in the same email.

```
$ openshift-install create install-config
? SSH Public Key /home/lab-user/.ssh/ocp.pub
? Platform vsphere
? vCenter vcenter.sddc-44-197-86-61.vmwarevmc.com
? Username sandbox-glnm2@vc.opentlc.com
? Password [? for help] ************
INFO Connecting to vCenter vcenter.sddc-44-197-86-61.vmwarevmc.com
INFO Defaulting to only available datacenter: SDDC-Datacenter
INFO Defaulting to only available cluster: /SDDC-Datacenter/host/Cluster-1
INFO Defaulting to only available datastore: /SDDC-Datacenter/datastore/WorkloadDatastore
INFO Defaulting to only available network: segment-sandbox-glnm2
? Virtual IP Address for API 192.168.95.201
? Virtual IP Address for Ingress 192.168.95.202
? Base Domain dynamic.opentlc.com
? Cluster Name glnm2
? Pull Secret [? for help] ***************************...
INFO Install-Config created in: .
```

The provided vCenter user does not have enough privileges to create its own folders, but one is created by environment provisioning system and communicated in the email from RHDP.  Edit the resulting install-config.yaml file and add the vcenter folder where the VMs will be created. Use the prefix /SDDC-Datacenter/vm/.
```
...
platform:
  vsphere:
	apiVIPs:
	- 192.168.95.201
	failureDomains:
	- name: generated-failure-domain
  	region: generated-region
  	server: vcenter.sddc-44-197-86-61.vmwarevmc.com
  	topology:
    	computeCluster: /SDDC-Datacenter/host/Cluster-1
    	datacenter: SDDC-Datacenter
    	datastore: /SDDC-Datacenter/datastore/WorkloadDatastore
    	networks:
    	- segment-sandbox-glnm2
    	resourcePool: /SDDC-Datacenter/host/Cluster-1//Resources
    	folder: /SDDC-Datacenter/vm/Workloads/sandbox-glnm2
  	zone: generated-zone
...
```
Create a directory with the name of the cluster in the bastion host and copy the install-config.yaml file there:
```
$ mkdir glnm2
$ cp install-config.yaml glnm2/
```

Run the installer
```
$ openshift-install create cluster --dir glnm2/
INFO Consuming Install Config from target directory
INFO Creating infrastructure resources...    	 
INFO Waiting up to 20m0s (until 3:21AM EST) for the Kubernetes API at https://api.glnm2.dynamic.opentlc.com:6443...
INFO API v1.28.6+6216ea1 up                  	 
INFO Waiting up to 1h0m0s (until 4:04AM EST) for bootstrapping to complete...
INFO Destroying the bootstrap resources...   	 
INFO Waiting up to 40m0s (until 3:58AM EST) for the cluster at https://api.glnm2.dynamic.opentlc.com:6443 to initialize...
INFO Waiting up to 30m0s (until 4:03AM EST) to ensure each cluster operator has finished progressing...
INFO All cluster operators have completed progressing
INFO Checking to see if there is a route at openshift-console/console...
INFO Install complete!                       	 
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/lab-user/glnm2/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.glnm2.dynamic.opentlc.com
INFO Login to the console with user: "kubeadmin", and password: "7azGi-RUI6u-HR8ph-PuRyH"
INFO Time elapsed: 36m40s
```

## Infra Nodes During Cluster Deployment

[Infra](https://access.redhat.com/solutions/5034771) nodes can be used to isolate infrastructure workloads for two primary purposes:

   1. To prevent incurring billing costs against subscription counts.
   1. To separate maintenance and management.

Infra nodes can be created during cluster deployment or post cluster deployment. 

To have infra nodes created at cluster deployment, start by creating the manifests from the install-config.yaml file.

```
$ openshift-install create manifests --dir glnm2/
INFO Consuming Install Config from target directory
INFO Manifests created in: glnm2/manifests and glnm2/openshift
```

One of the manifests defines the worker machineset
```
$ ls glnm2/openshift/*machineset*
glnm2/openshift/99_openshift-cluster-api_worker-machineset-0.yaml
```
### Adding An Infra Machine Set

In this case a new machineset for infra nodes is created, resulting in 2 machinesets: one for workers and one for infra. See later for an example in which the worker machineset is replaced by an infra machineset.

Copy the worker machineset as the infra machineset

```
$ cd glnm2/openshift/

$ cp 99_openshift-cluster-api_worker-machineset-0.yaml \
  99_openshift-cluster-api_infra-machineset-0.yaml
```
Modify the infra machineset according to the [documentation.](https://docs.openshift.com/container-platform/4.13/machine_management/creating-infrastructure-machinesets.html#machineset-yaml-vsphere_creating-infrastructure-machinesets)  Replace all references to worker and put infra instead except for for **userDataSecret** referencing the worker user data.

What makes the resulting nodes created from this machineset, infra nodes, is the label and toleration added to them:

```
...
spec:
  metadata:
    labels:
      node-role.kubernetes.io/infra: ""
    taints:
      - key: node-role.kubernetes.io/infra
        effect: NoSchedule
...
```

The amount of disk, CPU, and memory resources can be specified in the machineset definition. 

The final result looks like this:
```
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  creationTimestamp: null
  labels:
    machine.openshift.io/cluster-api-cluster: 6lzvs-xbd25
  name: 6lzvs-xbd25-infra-0
  namespace: openshift-machine-api
spec:
  replicas: 3
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: 6lzvs-xbd25
      machine.openshift.io/cluster-api-machineset: 6lzvs-xbd25-infra-0
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: 6lzvs-xbd25
        machine.openshift.io/cluster-api-machine-role: infra
        machine.openshift.io/cluster-api-machine-type: infra
        machine.openshift.io/cluster-api-machineset: 6lzvs-xbd25-infra-0
    spec:
      lifecycleHooks: {}
      metadata:
        labels:
          node-role.kubernetes.io/infra: ""
        taints:
          - key: node-role.kubernetes.io/infra
            effect: NoSchedule
      providerSpec:
        value:
          apiVersion: machine.openshift.io/v1beta1
          credentialsSecret:
            name: vsphere-cloud-credentials
          diskGiB: 120
          kind: VSphereMachineProviderSpec
          memoryMiB: 16384
          metadata:
            creationTimestamp: null
          network:
            devices:
            - networkName: segment-sandbox-6lzvs
          numCPUs: 4
          numCoresPerSocket: 4
          snapshot: ""
          template: 6lzvs-xbd25-rhcos-generated-region-generated-zone
          userDataSecret:
            name: worker-user-data
          workspace:
            datacenter: SDDC-Datacenter
            datastore: /SDDC-Datacenter/datastore/WorkloadDatastore
            folder: /SDDC-Datacenter/vm/Workloads/sandbox-6lzvs
            resourcePool: /SDDC-Datacenter/host/Cluster-1//Resources
            server: vcenter.sddc-44-197-86-61.vmwarevmc.com
```

Run the installer.
```
$ openshift-install create cluster --dir glnm2/
INFO Consuming OpenShift Install (Manifests) from target directory 
INFO Consuming Master Machines from target directory 
INFO Consuming Common Manifests from target directory 
INFO Consuming Worker Machines from target directory 
INFO Consuming Openshift Manifests from target directory 
INFO Creating infrastructure resources...         
INFO Waiting up to 20m0s (until 4:49AM EST) for the Kubernetes API at https://api.glnm2.dynamic.opentlc.com:6443... 
INFO API v1.28.6+6216ea1 up                       
INFO Waiting up to 1h0m0s (until 5:32AM EST) for bootstrapping to complete... 
INFO Destroying the bootstrap resources...        
INFO Waiting up to 40m0s (until 5:29AM EST) for the cluster at https://api.glnm2.dynamic.opentlc.com:6443 to initialize... 
INFO Waiting up to 30m0s (until 5:32AM EST) to ensure each cluster operator has finished progressing... 
INFO All cluster operators have completed progressing 
INFO Checking to see if there is a route at openshift-console/console... 
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/lab-user/glnm2/auth/kubeconfig' 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.glnm2.dynamic.opentlc.com 
INFO Login to the console with user: "kubeadmin", and password: "RLAiW-c83BZ-9SiAU-89pWa" 
INFO Time elapsed: 38m14s
```
The resulting cluster has 3 worker and 3 infra nodes, but the infra services are running on the worker nodes because those infra services don’t have the matching tolerations to run on the infra nodes.  This must be taken into account if the cluster does not have any worker nodes.

```
$ oc get nodes
NAME                         STATUS   ROLES                  AGE   VERSION
6lzvs-xbd25-master-0         Ready    control-plane,master   35m   v1.28.6+6216ea1
6lzvs-xbd25-master-1         Ready    control-plane,master   35m   v1.28.6+6216ea1
6lzvs-xbd25-master-2         Ready    control-plane,master   34m   v1.28.6+6216ea1
6lzvs-xbd25-infra-0-75cmf    Ready    infra,worker           16m   v1.28.6+6216ea1
6lzvs-xbd25-infra-0-kbfrh    Ready    infra,worker           16m   v1.28.6+6216ea1
6lzvs-xbd25-infra-0-rm524    Ready    infra,worker           16m   v1.28.6+6216ea1
6lzvs-xbd25-worker-0-ngrlc   Ready    worker                 16m   v1.28.6+6216ea1
6lzvs-xbd25-worker-0-np8q8   Ready    worker                 16m   v1.28.6+6216ea1
6lzvs-xbd25-worker-0-t8lml   Ready    worker                 16m   v1.28.6+6216ea1

$ oc get machineset -n openshift-machine-api
NAME                   DESIRED   CURRENT   READY   AVAILABLE   AGE
6lzvs-xbd25-infra-0    3         3         3       3           39m
6lzvs-xbd25-worker-0   3         3         3       3           39m
```

#### Assigning Infra Workloads to Infra Nodes

Infra workloads are running on the worker nodes because, out of the box, they lack the neccessary tolerations to run on the infra nodes.  In the next example, the router pods are running in worker nodes:

```
$ oc get pods -n openshift-ingress -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP               NODE                         NOMINATED NODE   READINESS GATES
router-default-54f47c8bb9-748m7   1/1     Running   0          26m   192.168.74.151   6lzvs-qrx2p-worker-0-ljzpg   <none>           <none>
router-default-54f47c8bb9-c27p7   1/1     Running   0          26m   192.168.74.80    6lzvs-qrx2p-worker-0-l2x7r   <none>           <none>
```
To **migrate the router pods** so they run on the infra nodes, edit the default ingress controller and add a nodePlacement section that matches the infra label.  Add the toleration for the infra taint, the toleration value is left out because no value was assigned to the taint in the infra nodes:

```
...
spec:
  clientTLS:
    clientCA:
      name: ""
    clientCertificatePolicy: ""
  httpCompression: {}
  httpEmptyRequestsPolicy: Respond
  httpErrorCodePages:
    name: ""
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/infra: ""
    tolerations:
    - effect: NoSchedule
      key: node-role.kubernetes.io/infra
  replicas: 2
  tuningOptions:
    reloadInterval: 0s
...
```
After the change is applied, the router pods are redeployed to the infra nodes.
```
$ oc get pods -n openshift-ingress -o wide                                                                                                                         
NAME                              READY   STATUS        RESTARTS   AGE   IP               NODE                         NOMINATED NODE   READINESS GATES                                      
router-default-54f47c8bb9-c27p7   1/1     Terminating   0          36m   192.168.74.80    6lzvs-qrx2p-worker-0-l2x7r   <none>           <none>                                               
router-default-644695d789-l8q68   1/1     Running       0          51s   192.168.74.152   6lzvs-qrx2p-infra-0-6zh5b    <none>           <none>                                               
router-default-644695d789-rrxrv   1/1     Running       0          15s   192.168.74.150   6lzvs-qrx2p-infra-0-gzmwd    <none>           <none>
```
To **migrate the registry pods to the infra nodes** [add a nodePlacement section to the registry config object](https://docs.openshift.com/container-platform/4.13/machine_management/creating-infrastructure-machinesets.html#infrastructure-moving-registry_creating-infrastructure-machinesets).

In the case of a vsphere IPI installation, the registry is in a removed management state because no object storage is available:

```
$ oc get config cluster -o yaml
apiVersion: imageregistry.operator.openshift.io/v1
kind: Config
metadata:
  name: cluster
spec:
  logLevel: Normal
  managementState: Removed
  observedConfig: null
  operatorLogLevel: Normal
  proxy: {}
  replicas: 1
  requests:
    read:
      maxWaitInQueue: 0s
    write:
      maxWaitInQueue: 0s
  rolloutStrategy: RollingUpdate
  storage: {}
```
This has the effect of not having any registry pods running in the cluster:
```
$ oc get pod -n openshift-image-registry -l docker-registry=default
No resources found in openshift-image-registry namespace.
```

To assign some storage to the registry, create a PVC object like the following, of type ReadWriteOnce which is the mode available from the thin CSI driver in vSphere.  A reference yaml file can be found in this repository at **IPI/image-registry-storage-pvc.yaml**:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-registry-storage
  namespace: openshift-image-registry
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: thin-csi
  volumeMode: Filesystem

$ oc create -f image-registry-storage.yaml
persistentvolumeclaim/image-registry-storage created
```

Set the management state to Managed and add a storage section.  In this case the storage is provided by the PVC created above which only creates RWO PVCs, so only one replica of the registry pod can be running at any given time.  

Add a node selector to pin the registry to the infra nodes, and the toleration to allow the pod to be executed there.  The toleration value is left out because no value was assigned to the taint in the infra nodes

Set the rolloutStrategy to Recreate.  This is to prevent blocking the redeployment of the registry pod in case a change in configuration is made, if Recreate is not used, the running pod does not release the PVC and the new pod cannot be created.  A reference yaml file with the configuration can be found in this repository at **IPI/config-image-registry.yaml**:
```
apiVersion: imageregistry.operator.openshift.io/v1
kind: Config
metadata:
  name: cluster
spec:
  logLevel: Normal
  managementState: Managed
  observedConfig: null
  operatorLogLevel: Normal
  proxy: {}
  replicas: 1
  requests:
    read:
      maxWaitInQueue: 0s
    write:
      maxWaitInQueue: 0s
  rolloutStrategy: Recreate
  storage:
    managementState: Managed
    pvc:
      claim: image-registry-storage
  unsupportedConfigOverrides: null
  nodeSelector:
    node-role.kubernetes.io/infra: ""
  tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/infra
```
The new registry pod is started in an infra node:
```
$ oc get pod -n openshift-image-registry -l docker-registry=default -o wide
NAME                              READY   STATUS    RESTARTS   AGE     IP           NODE                        NOMINATED NODE   READINESS GATES
image-registry-5cd7cbb669-7kv54   1/1     Running   0          6m52s   10.131.2.7   6lzvs-qrx2p-infra-0-6zh5b   <none>           <none>
```

To **migrate the monitoring stack to the infra nodes** create a configmap in the openshift-monitoring namespace with the nodePlacement section and toleratoins for all the monitoring components. A reference yaml file can be found in this repository at IPI/cluster-monitoring-config-map:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |+
    alertmanagerMain:
      nodeSelector: 
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        effect: NoSchedule
    prometheusK8s:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        effect: NoSchedule
    prometheusOperator:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        effect: NoSchedule
    k8sPrometheusAdapter:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        effect: NoSchedule
    kubeStateMetrics:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        effect: NoSchedule
    telemeterClient:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        effect: NoSchedule
    openshiftStateMetrics:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        effect: NoSchedule
    thanosQuerier:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        effect: NoSchedule
```
Apply the config map definition
```
$ oc create -f cluster-monitoring-config-map                                                                                                                       
configmap/cluster-monitoring-config created

$ oc get pods -n openshift-monitoring -o wide
NAME                                                     READY   STATUS    RESTARTS   AGE    IP               NODE                         NOMINATED NODE   READINESS GATES
alertmanager-main-0                                      6/6     Running   0          43s    10.129.2.15      6lzvs-qrx2p-infra-0-gzmwd    <none>           <none>
alertmanager-main-1                                      6/6     Running   0          76s    10.128.2.12      6lzvs-qrx2p-infra-0-vlwr5    <none>           <none>
cluster-monitoring-operator-6c76c86bcc-l4dn5             1/1     Running   0          152m   10.128.0.9       6lzvs-qrx2p-master-0         <none>           <none>
kube-state-metrics-7f74f69554-qmpmw                      3/3     Running   0          81s    10.131.2.9       6lzvs-qrx2p-infra-0-6zh5b    <none>           <none>
monitoring-plugin-647d586488-mtfnj                       1/1     Running   0          133m   10.131.0.11      6lzvs-qrx2p-worker-0-rrb5k   <none>           <none>
monitoring-plugin-647d586488-s5t6v                       1/1     Running   0          133m   10.129.2.7       6lzvs-qrx2p-infra-0-gzmwd    <none>           <none>
node-exporter-2cskx                                      2/2     Running   0          133m   192.168.74.79    6lzvs-qrx2p-infra-0-vlwr5    <none>           <none>
node-exporter-2llz2                                      2/2     Running   0          133m   192.168.74.152   6lzvs-qrx2p-infra-0-6zh5b    <none>           <none>
node-exporter-2nn42                                      2/2     Running   0          133m   192.168.74.80    6lzvs-qrx2p-worker-0-l2x7r   <none>           <none>
node-exporter-89hlf                                      2/2     Running   0          133m   192.168.74.150   6lzvs-qrx2p-infra-0-gzmwd    <none>           <none>
node-exporter-8pb6n                                      2/2     Running   0          133m   192.168.74.149   6lzvs-qrx2p-worker-0-rrb5k   <none>           <none>
node-exporter-k5ps6                                      2/2     Running   0          133m   192.168.74.147   6lzvs-qrx2p-master-0         <none>           <none>
node-exporter-pt6q6                                      2/2     Running   0          133m   192.168.74.148   6lzvs-qrx2p-master-2         <none>           <none>
node-exporter-r2jms                                      2/2     Running   0          133m   192.168.74.151   6lzvs-qrx2p-worker-0-ljzpg   <none>           <none>
node-exporter-wbplf                                      2/2     Running   0          133m   192.168.74.44    6lzvs-qrx2p-master-1         <none>           <none>
openshift-state-metrics-db6888695-dhsj4                  3/3     Running   0          81s    10.131.2.10      6lzvs-qrx2p-infra-0-6zh5b    <none>           <none>
prometheus-adapter-6d95cc7bd9-jlswd                      1/1     Running   0          78s    10.129.2.13      6lzvs-qrx2p-infra-0-gzmwd    <none>           <none>
prometheus-adapter-6d95cc7bd9-q5hst                      1/1     Running   0          78s    10.131.2.12      6lzvs-qrx2p-infra-0-6zh5b    <none>           <none>
prometheus-k8s-0                                         6/6     Running   0          46s    10.128.2.13      6lzvs-qrx2p-infra-0-vlwr5    <none>           <none>
prometheus-k8s-1                                         6/6     Running   0          65s    10.129.2.14      6lzvs-qrx2p-infra-0-gzmwd    <none>           <none>
prometheus-operator-7d84c9789-95tz5                      2/2     Running   0          88s    10.129.2.11      6lzvs-qrx2p-infra-0-gzmwd    <none>           <none>
prometheus-operator-admission-webhook-667fbc8566-ts22d   1/1     Running   0          93s    10.131.2.8       6lzvs-qrx2p-infra-0-6zh5b    <none>           <none>
prometheus-operator-admission-webhook-667fbc8566-vc42q   1/1     Running   0          93s    10.129.2.10      6lzvs-qrx2p-infra-0-gzmwd    <none>           <none>
telemeter-client-b69cb8cbd-fn5v6                         3/3     Running   0          80s    10.129.2.12      6lzvs-qrx2p-infra-0-gzmwd    <none>           <none>
thanos-querier-858b5894d-995gh                           6/6     Running   0          78s    10.131.2.11      6lzvs-qrx2p-infra-0-6zh5b    <none>           <none>
thanos-querier-858b5894d-fzzvr                           6/6     Running   0          79s    10.128.2.11      6lzvs-qrx2p-infra-0-vlwr5    <none>           <none>
```
Monitoring-plugin pods are still running on worker nodes, this could be a bug in the monitoring operator.  The node exporters running on the workers is expected, one node exporter should be running on each and every node.

Some other "default" infra workloads continue running in the worker nodes.  This is one of the reasons why the taint is left out when the cluster does not have worker nodes, only infra and masters, to ensure these pods can still run on the infra nodes:

 * Cronjobs for image-prunner 
 * openshift-operator-lifecycle-manager may need to be modified so that the pods can run on infra nodes.
 * The network-check-source pod 
```
$ oc get po -n openshift-network-diagnostics -o wide -l app=network-check-source
NAME                                   READY   STATUS    RESTARTS       AGE    IP           NODE                         NOMINATED NODE   READINESS GATES
network-check-source-c9468c84c-w2hcf   1/1     Running   4 (140m ago)   155m   10.130.2.7   6lzvs-qrx2p-worker-0-ljzpg   <none>           <none>
```

### Replacing The Worker Machine Set

In case of a cluster in which no worker nodes are needed because only infra workloads are going to be executed, replace the worker machineset by an infra machineset and leave the taint out of the infra nodes.

After creating the manifests as described in section [Adding An Infra Machine Set](#adding-an-infra-machine-set), rename the file defining the worker machineset and edit it.    Keep the original value for **userDataSecret** referencing the worker user data.

The machineset definition does not contain any [taints](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/).  If there were any taints with the NoSchedule or NoExecute effect, the installation would not complete successfully because the infrastructure worloads created during installation don't have the corresponding tolerations.  The tainst are not needed because this cluster does not have any worker nodes and therefore no user workloads are expected here.

```
$ mv 99_openshift-cluster-api_worker-machineset-0.yaml \
   99_openshift-cluster-api_infra-machineset-0.yaml

$ vim 99_openshift-cluster-api_infra-machineset-0.yaml
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  creationTimestamp: null
  labels:
    machine.openshift.io/cluster-api-cluster: 6lzvs-2czxc
  name: 6lzvs-2czxc-infra-0
  namespace: openshift-machine-api
spec:
  replicas: 3
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: 6lzvs-2czxc
      machine.openshift.io/cluster-api-machineset: 6lzvs-2czxc-infra-0
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: 6lzvs-2czxc
        machine.openshift.io/cluster-api-machine-role: infra
        machine.openshift.io/cluster-api-machine-type: infra
        machine.openshift.io/cluster-api-machineset: 6lzvs-2czxc-infra-0
    spec:
      lifecycleHooks: {}
      metadata:
        labels:
          node-role.kubernetes.io/infra: ""
      providerSpec:
        value:
          apiVersion: machine.openshift.io/v1beta1
          credentialsSecret:
            name: vsphere-cloud-credentials
          diskGiB: 120
          kind: VSphereMachineProviderSpec
          memoryMiB: 16384
          metadata:
            creationTimestamp: null
          network:
            devices:
            - networkName: segment-sandbox-6lzvs
          numCPUs: 4
          numCoresPerSocket: 4
          snapshot: ""
          template: 6lzvs-2czxc-rhcos-generated-region-generated-zone
          userDataSecret:
            name: worker-user-data
          workspace:
            datacenter: SDDC-Datacenter
            datastore: /SDDC-Datacenter/datastore/WorkloadDatastore
            folder: /SDDC-Datacenter/vm/Workloads/sandbox-6lzvs
            resourcePool: /SDDC-Datacenter/host/Cluster-1//Resources
            server: vcenter.sddc-44-197-86-61.vmwarevmc.com
```
Run the installer.
```
$ openshift-install create cluster --dir 6lzvs/
INFO Consuming OpenShift Install (Manifests) from target directory 
INFO Consuming Common Manifests from target directory 
INFO Consuming Openshift Manifests from target directory 
INFO Consuming Worker Machines from target directory 
INFO Consuming Master Machines from target directory 
INFO Creating infrastructure resources...         
INFO Waiting up to 20m0s (until 3:49PM EDT) for the Kubernetes API at https://api.6lzvs.dynamic.opentlc.com:6443... 
INFO API v1.28.6+6216ea1 up                       
INFO Waiting up to 1h0m0s (until 4:32PM EDT) for bootstrapping to complete... 
INFO Destroying the bootstrap resources...        
INFO Waiting up to 40m0s (until 4:27PM EDT) for the cluster at https://api.6lzvs.dynamic.opentlc.com:6443 to initialize... 
INFO Waiting up to 30m0s (until 4:32PM EDT) to ensure each cluster operator has finished progressing... 
INFO All cluster operators have completed progressing 
INFO Checking to see if there is a route at openshift-console/console... 
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/lab-user/6lzvs/auth/kubeconfig' 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.6lzvs.dynamic.opentlc.com 
INFO Login to the console with user: "kubeadmin", and password: "ByXfw-psGGg-j6nba-F55Pd" 
INFO Time elapsed: 37m8s
```
The deployed cluster has 3 masters and 3 infras, no workers, and a single machineset for the infra nodes.
```
$ oc get nodes
NAME                        STATUS   ROLES                  AGE   VERSION
6lzvs-2czxc-infra-0-jpg2w   Ready    infra,worker           10h   v1.28.6+6216ea1
6lzvs-2czxc-infra-0-vclbd   Ready    infra,worker           10h   v1.28.6+6216ea1
6lzvs-2czxc-infra-0-x77tg   Ready    infra,worker           10h   v1.28.6+6216ea1
6lzvs-2czxc-master-0        Ready    control-plane,master   11h   v1.28.6+6216ea1
6lzvs-2czxc-master-1        Ready    control-plane,master   11h   v1.28.6+6216ea1
6lzvs-2czxc-master-2        Ready    control-plane,master   11h   v1.28.6+6216ea1

$ oc get machineset -n openshift-machine-api
NAME                  DESIRED   CURRENT   READY   AVAILABLE   AGE
6lzvs-2czxc-infra-0   3         3         3       3           11h
```
The infra workloads are running in the infra nodes because there are not taints assigned to those nodes, so tolerations are not required:
```
$ oc get pods -n openshift-ingress -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP               NODE                        NOMINATED NODE   READINESS GATES
router-default-54f47c8bb9-vddkp   1/1     Running   0          11h   192.168.74.145   6lzvs-2czxc-infra-0-vclbd   <none>           <none>
router-default-54f47c8bb9-zjlpr   1/1     Running   0          11h   192.168.74.77    6lzvs-2czxc-infra-0-jpg2w   <none>           <none>
```

## Installing RHACM on Infra Nodes

This section describes how to install RHACM to run its components on infra nodes, the oc CLI must be used for this, the installation instructions are based on the following chapters from the RHACM documentation:

* [Installing from the OpenShift Container Platform CLI](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.9/html/install/installing#installing-from-the-cli)
* [Installing the Red Hat Advanced Cluster Management hub cluster on infrastructure nodes](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.9/html/install/installing#installing-on-infra-node)

These instructions are the same no matter if the cluster is connected or disconnected.

Create the recommended namespace to install ACM components:

```
$ oc create namespace open-cluster-management
namespace/open-cluster-management created

$ oc project open-cluster-management
Now using project "open-cluster-management" on server "https://api.6lzvs.dynamic.opentlc.com:6443".
```

Create the operator group based on the following yaml definition.  A reference yaml file can be found in this repository at **IPI/ACM/og.yaml**
```
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: rhacm-og
  namespace: open-cluster-management
spec:
  targetNamespaces:
  - open-cluster-management
```

```
$ oc create -f og.yaml 
operatorgroup.operators.coreos.com/rhacm-og created
```

Create the operator subscription based on the following yaml definition. A reference yaml file can be found in this repository at **IPI/ACM/subs.yaml**.

The node selector and tolerations ensure that the RHACM components are started on the infra nodes.  The RHACM components can still run on the infra nodes even if those nodes don't have the corresponding taint.

```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: acm-operator-subscription
  namespace: open-cluster-management
spec:
  sourceNamespace: openshift-marketplace
  source: redhat-operators
  channel: release-2.9
  installPlanApproval: Automatic
  name: advanced-cluster-management
  config:
    nodeSelector:
      node-role.kubernetes.io/infra: ""
    tolerations:
    - key: node-role.kubernetes.io/infra
      effect: NoSchedule
      operator: Exists
```

Create the subscription object.  The cluster may take a minute to react to the new subscription:
```
$ oc create -f subs.yaml 
subscription.operators.coreos.com/acm-operator-subscription created

$ oc get po
NAME                                        READY   STATUS    RESTARTS   AGE
multiclusterhub-operator-57c85fd5b9-pn52l   1/1     Running   0          46s

$ oc get csv
NAME                                 DISPLAY                                      VERSION   REPLACES                             PHASE
advanced-cluster-management.v2.9.3   Advanced Cluster Management for Kubernetes   2.9.3     advanced-cluster-management.v2.9.2   Succeeded
```

Create the multiclusterhub custom resource.  The yaml definition changes slightly depending on the environment connection status:
* In a disconnected environment a special annotation needs to be added to the multiclusterhub definition with the name of the catalogsource for the red hat operators.
* In a connected environment the annotation is not used.

Reference yaml files can be found in this repository at **IPI/ACM/multiclusterhub.yaml** and **IPI/ACM/multiclusterhub-disconnected.yaml**
```
apiVersion: operator.open-cluster-management.io/v1
kind: MultiClusterHub
metadata:
  name: multiclusterhub
  namespace: open-cluster-management
  annotations:
    installer.open-cluster-management.io/mce-subscription-spec: '{"source": "cs-redhat-operator-index"}'
spec:
  nodeSelector:
    node-role.kubernetes.io/infra: ""
```
Create the multicluster hub.  It takes a few minutes for all the pods to be started, after starting a few pods it looks like nothing is happenning, but after a couple minutes the rest of the pods are started:
```
$ oc create -f multiclusterhub.yaml                                                                                                                                
multiclusterhub.operator.open-cluster-management.io/multiclusterhub created

$ oc get pod -o wide
NAME                                                              READY   STATUS    RESTARTS        AGE     IP            NODE                        NOMINATED NODE   READINESS GATES
cluster-permission-776865b85c-tk2js                               1/1     Running   0               5m16s   10.129.2.37   6lzvs-qrx2p-infra-0-gzmwd   <none>           <none>
console-chart-console-v2-566dbfb5c6-cr4x6                         1/1     Running   0               5m15s   10.131.2.41   6lzvs-qrx2p-infra-0-6zh5b   <none>           <none>
console-chart-console-v2-566dbfb5c6-rgphk                         1/1     Running   0               5m14s   10.129.2.38   6lzvs-qrx2p-infra-0-gzmwd   <none>           <none>
grc-policy-addon-controller-7896846658-h97vr                      1/1     Running   0               5m13s   10.131.2.42   6lzvs-qrx2p-infra-0-6zh5b   <none>           <none>
grc-policy-addon-controller-7896846658-xjrcv                      1/1     Running   0               5m13s   10.129.2.39   6lzvs-qrx2p-infra-0-gzmwd   <none>           <none>
grc-policy-propagator-6444bcd8bb-frg2j                            2/2     Running   0               5m11s   10.129.2.40   6lzvs-qrx2p-infra-0-gzmwd   <none>           <none>
grc-policy-propagator-6444bcd8bb-zldrd                            2/2     Running   0               5m11s   10.131.2.43   6lzvs-qrx2p-infra-0-6zh5b   <none>           <none>
insights-client-7769f5fbf-tk2sr                                   1/1     Running   0               5m9s    10.131.2.44   6lzvs-qrx2p-infra-0-6zh5b   <none>           <none>
insights-metrics-8ff75bbcb-kwkl4                                  2/2     Running   0               5m10s   10.128.2.32   6lzvs-qrx2p-infra-0-vlwr5   <none>           <none>
klusterlet-addon-controller-v2-759678d549-5n9wp                   1/1     Running   0               5m16s   10.131.2.40   6lzvs-qrx2p-infra-0-6zh5b   <none>           <none>
klusterlet-addon-controller-v2-759678d549-hfjqt                   1/1     Running   0               5m16s   10.129.2.36   6lzvs-qrx2p-infra-0-gzmwd   <none>           <none>
multicluster-integrations-5cfb9ddc47-knp84                        3/3     Running   1 (9m31s ago)   11m     10.131.2.14   6lzvs-qrx2p-infra-0-6zh5b   <none>           <none>
multicluster-observability-operator-849fc87b67-fb98n              1/1     Running   0               5m8s    10.131.2.45   6lzvs-qrx2p-infra-0-6zh5b   <none>           <none>
multicluster-operators-application-589667688b-t8zlv               3/3     Running   2 (9m30s ago)   11m     10.129.2.16   6lzvs-qrx2p-infra-0-gzmwd   <none>           <none>
multicluster-operators-channel-698bf9dbbb-hkppf                   1/1     Running   1 (9m25s ago)   11m     10.131.2.15   6lzvs-qrx2p-infra-0-6zh5b   <none>           <none>
multicluster-operators-hub-subscription-86b5c8876d-4dgk8          1/1     Running   1 (9m26s ago)   11m     10.128.2.16   6lzvs-qrx2p-infra-0-vlwr5   <none>           <none>
multicluster-operators-standalone-subscription-5757b64469-sdlx9   1/1     Running   0               11m     10.128.2.14   6lzvs-qrx2p-infra-0-vlwr5   <none>           <none>
multicluster-operators-subscription-report-85cf8979c-rk9xz        1/1     Running   0               11m     10.128.2.15   6lzvs-qrx2p-infra-0-vlwr5   <none>           <none>
multiclusterhub-operator-57c85fd5b9-pn52l                         1/1     Running   0               23m     10.131.2.13   6lzvs-qrx2p-infra-0-6zh5b   <none>           <none>
search-api-5c5c7b95b6-42vnb                                       1/1     Running   0               4m54s   10.131.2.49   6lzvs-qrx2p-infra-0-6zh5b   <none>           <none>
search-collector-6bd977c768-db76f                                 1/1     Running   0               4m54s   10.131.2.48   6lzvs-qrx2p-infra-0-6zh5b   <none>           <none>
search-indexer-6444885d8-frfhx                                    1/1     Running   0               4m54s   10.129.2.44   6lzvs-qrx2p-infra-0-gzmwd   <none>           <none>
search-postgres-7dd8974cc8-flt6t                                  1/1     Running   0               4m55s   10.131.2.47   6lzvs-qrx2p-infra-0-6zh5b   <none>           <none>
search-v2-operator-controller-manager-5dcbb5c8b4-dtfls            2/2     Running   0               5m8s    10.129.2.41   6lzvs-qrx2p-infra-0-gzmwd   <none>           <none>
submariner-addon-dcf4dcb9c-rlp7r                                  1/1     Running   0               5m7s    10.128.2.33   6lzvs-qrx2p-infra-0-vlwr5   <none>           <none>
volsync-addon-controller-5cd7589555-hklv5                         1/1     Running   0               5m6s    10.131.2.46   6lzvs-qrx2p-infra-0-6zh5b   <none>           <none>
```
A new dropdown menu appears in the Openshift web site.
![ACM Drop Down Menu](images/image29.png)

The local cluster should go into status Ready after a couple minutes:
![Status Ready for Local Cluster](images/image44.png)

## Disconnected Environments

This section describes how to deploy an Openshift cluster on a disconnected environment, considering the special limitations at [demo.redhat.com](https://demo.redhat.com).

The disconnected environment in this example does not have direct access to the public Red Hat image registries in order to fetch the images required for the installation.  A local registry must be provisioned so the cluster nodes can download the images from it and the installation can succeed. 

The local registry used is the [quay mirror registry](https://github.com/quay/mirror-registry)

The mirror registry service name needs to be resolvable by DNS, from any client accessing it.  Considering demo.redhat.com limitations regarding DNS in demo.redhat.com vSphere based environments, in which it is not possible to create new or modify existing DNS records, we are going to deploy the mirror registry remotely from one VMware Cloud Open Environment onto another so the bastion host on the second environment can be used as the mirror registry for the first or any other environment.

The bastion host public DNS name in the second environment will be used to access the mirror registry.

![Mirror Registry Across Environments](images/MirrorRegistryAcross.drawio.png)

### Setting Up the Environment

Create a new VMware Cloud Public Cloud Open Environment.

On the new environment, the bastion host does not have enough resources to run the mirror registry. 

Shutdown the bastion host and add:

* One additional vCPU, 2 in total
* Increase the memory to 8GB
* Add a disk of 250GB.

![Extend Bastion Resources](images/BastionMirrorResources.png)

Boot up the bastion.

Ssh into the bastion host and show the available disks, a new 250GB sdb disk should appear in the output:
```
$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   30G  0 disk 
├─sda1   8:1    0    1M  0 part 
├─sda2   8:2    0  100M  0 part /boot/efi
└─sda3   8:3    0 29.9G  0 part /
sdb      8:16   0  250G  0 disk 
sr0     11:0    1 1024M  0 rom
```
Use LVM instead of the raw disk device, this way the partition can be extend if you run out of space for the mirror registry.
```
$ sudo yum install -y lvm2

$ sudo pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.

$ sudo vgcreate vgmirror /dev/sdb
  Volume group "vgmirror" successfully created

$ sudo vgs
  VG       #PV #LV #SN Attr   VSize    VFree   
  vgmirror   1   0   0 wz--n- <250.00g <250.00g

$ sudo lvcreate -n lvmirror -l +100%FREE vgmirror
  Logical volume "lvmirror" created.

$ sudo lvs
  LV       VG       Attr       LSize    Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lvmirror vgmirror -wi-a----- <250.00g 

$ sudo mkfs.xfs /dev/vgmirror/lvmirror
```

Create the mount point
```
$ sudo mkdir /var/mirror-registry
```

Get the UUID for the /dev/sdb disk
```
$ sudo blkid
...
/dev/mapper/vgmirror-lvmirror: UUID="0ce458b9-0192-43d2-9cd1-beed7900c075" BLOCK_SIZE="512" TYPE="xfs"
```

Add an entry like the following to the /etc/fstab file
```
UUID=d327e61d-8372-435...   /var/mirror-registry   xfs  defaults  0  0
```

Mount the partition
```
$ sudo mount /var/mirror-registry/

$ df -Ph
Filesystem                     Size  Used Avail Use% Mounted on
...
/dev/mapper/vgmirror-lvmirror  250G  1.8G  249G   1% /var/mirror-registry
```

### Installing the Mirror Registry

This instructions are base on [the official documentation](https://docs.openshift.com/container-platform/4.15/installing/disconnected_install/installing-mirroring-creating-registry.html#mirror-registry-localhost_installing-mirroring-creating-registry)

The mirror registry installer is downloaded and executed on the VMWare Open Environment 1, but the installation target is the VMWare Open Environment 2 where the actual mirror registry will run.

The mirror registry installer is executed on the first environment, but it needs to run an ansible playbook with root privileges against the second environment so that the registry service can listen on privileged port 443.  

Copy the existing ssh key for the root user in the second environment to the first environment:

```
<second env>$ sudo ls /root/.ssh/
authorized_keys  bastion_vhnwk  bastion_vhnwk.pub  config

<second env>$ sudo cat /root/.ssh/bastion_vhnwk
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEA1QlqAIineSlrxIyEgOQ1A78uD36e1p32sTtpqCV/O0ZUWng1LBhz
...
9yijev/D9OldJLAAAAEnJvb3RAYmFzdGlvbi05OTVwdg==
-----END OPENSSH PRIVATE KEY-----

<first env>$ vim ~/.ssh/mirror-root
<first env>$ chmod 0400 ~/.ssh/mirror-root
<first env>$ ssh -i ~/.ssh/mirror-root root@bastion-995pv.995pv.dynamic.opentlc.com
Last failed login: Fri Mar  8 04:31:40 EST 2024 from 79.138.85.216 on ssh:notty
There were 17 failed login attempts since the last successful login.
Last login: Thu Dec  7 05:08:51 2023

<first env># whoami
root
```

> If a non privileged port like the default 8443 is used for the mirror registry service, the ansible playbook can be run as a normal user in the second environment and an ssh key for the lab user can be created and copied over.  In this case the unprivileged user must be able to write to the quayStorage directory.
```
<first env>$ ssh-keygen -N '' -f ~/.ssh/mirror-unprivileged
<first env>$ ssh-copy-id -i ~/.ssh/mirror.pub lab-user@bastion-995pv.995pv.dynamic.opentlc.com

<first env>$ ssh -i ~/.ssh/mirror lab-user@bastion-995pv.995pv.dynamic.opentlc.com
```
Download the **mirror-registry.tar.gz**, **oc mirror plugin** and the **oc** CLI packages from the [Red Hat Hybrid Cloud Console Downloads page](https://console.redhat.com/openshift/downloads).
![Download Mirror Registry Installer](images/DownloadMirrorRegistry.png)

Copy the files to the first environment and uncompress the tar file
```
$ scp mirror-registry.tar.gz oc-mirror.tar.gz openshift-client-linux.tar.gz lab-user@bastion-gvsfj.gvsfj.dynamic.opentlc.com:
lab-user@bastion-gvsfj.gvsfj.dynamic.opentlc.com's password:
mirror-registry.tar.gz                                                       100%  674MB  19.9MB/s   00:33
oc-mirror.tar.gz                                                             100%   62MB  19.4MB/s   00:03
openshift-client-linux.tar.gz                                                100%   62MB  19.1MB/s   00:03

<first env>$ mkdir mirror
<first env>$ mv mirror-registry.tar.gz oc-mirror.tar.gz mirror
<first env>$ cd mirror/
<first env>$ tar xvf mirror-registry.tar.gz 
image-archive.tar
execution-environment.tar
mirror-registry
```

Run the installer on the first env to install the mirror registry remotely on the second env.

The variable **--quayHostname** gets assigned the DNS name for the mirror registry service is the bastion's public hostname on the second environment and the port where the service listens on for requests, if no port is specified the default is 8443, the protocol is HTTPS regardless of the port.

`--quayHostname bastion-995pv.995pv.dynamic.opentlc.com:443`

The variable **--quayStorage** gets assigned a directory inside the partition created earlier, where the images will be storaged

`--quayStorage /var/mirror-registry/data`

Because this is a remote installation from the first environment to the second environment, the connection parameters for the second environment are needed:

 * If the mirror registry listens on port 443, because this is a privileged port the remote user must be root:

	`--targetHostname bastion-995pv.995pv.dynamic.opentlc.com --targetUsername root -k ~/.ssh/mirror-root`

 * If the mirror registry listens on port 8443 or other non privileged port the remote user can be lab-user:

	`--targetHostname bastion-995pv.995pv.dynamic.opentlc.com --targetUsername lab-user -k ~/.ssh/mirror-unprivileged`

Run the installer command:
```
<first env>$ ./mirror-registry install --quayHostname bastion-vhnwk.vhnwk.dynamic.opentlc.com:443 --quayStorage /var/mirror-registry/data --targetHostname bastion-vhnwk.vhnwk.dynamic.opentlc.com --targetUsername root -k ~/.ssh/mirror-root
                                                                                                                                                                                                 __   __                                                                                                                                                                                    
  /  \ /  \     ______   _    _     __   __   __                                                                                                                                               / /\ / /\ \   /  __  \ | |  | |   /  \  \ \ / /       
/ /  / /  \ \  | |  | | | |  | |  / /\ \  \   /                                                                                                                                               \ \  \ \  / /  | |__| | | |__| | / ____ \  | |                                                                                                                                                
 \ \/ \ \/ /   \_  ___/  \____/ /_/    \_\ |_|          
  \__/ \__/      \ \__                                                                                                                                                                        
                  \___\ by Red Hat                                                                                                                                                            
 Build, Store, and Distribute your Containers                                                                                                                                                
                                                                                                                                                                                              
INFO[2024-03-25 12:15:17] Install has begun 
...
INFO[2024-03-25 12:21:54] Quay is available at https://bastion-vhnwk.vhnwk.dynamic.opentlc.com:8443 with credentials (init, idEh...bQyc9Rm813)
```

The final message contains the URL to access the registry and the user credentials.

Save the credentials for later use.

The installer creates systemd services for the different components:

```
<second env>$ systemctl status quay-app.service
<second env>$ systemctl status quay-postgres.service
<second env>$ systemctl status quay-redis.service 
<second env>$ systemctl status quay-pod.service
```
Test the access to the registry.  The option --tls-verify=false is used so that podman does not reject the unknown root CA in the mirror registry:
```
<first env>$ podman login --tls-verify=false -u init -p UAha4s...9giwEj0 bastion-vhnwk.vhnwk.dynamic.opentlc.com
Login Succeeded!
```

Access the web interface using the bastion external name (i.e. https://bastion-sjmzk.sjmzk.dynamic.opentlc.com)

![Mirror Registry Web Interface](images/image49.png)

The mirror registry is ready to be used.

### Uninstalling the Mirror Registry

To uninstall the mirror registry run a command like the following.  

The command is similar to the one used to install the mirror registry but in this case the **quayHostname** parameter is not used and the **quayStorage** parameter is replaced by **quayRoot**.

The uninstall command stops the systemd services, removes the services and deletes the directory specified with quayRoot, efectively deleting any mirrores packages in the host.  Removing the directory does not work if this is the root of a disk partition.
```
<first env>$ ./mirror-registry uninstall -v --quayRoot /var/mirror-registry/data --targetHostname bastion-vhnwk.vhnwk.dynamic.opentlc.com --targetUsername root -k ~/.ssh/mirror-root
```

If you don't want to remove the mirrored images, just leave out the quayRoot directory and answer **no** to the question: "**Are you sure want to delete quayRoot directory ~/quay-install and all storage data? [y/n]**"

### Mirroring Images

The [oc mirror plugin](https://github.com/openshift/oc-mirror) is used to mirror the images from the public Red Hat registries to the local mirror registry.  

The bastion host may not have enough resources to run the oc mirror commands, if that is the case shutdown the bastion and add:

* One additional vCPU, 2 in total
* Increase the memory to 8GB
* Add a disk of 250GB.

Download the oc mirror plugin and the oc CLI packages from the [Red Hat Hybrid Cloud Console Downloads page](https://console.redhat.com/openshift/downloads).

Install the oc CLI and the oc mirror plugin in the first environment:
```
<first env>$ tar xvf oc-mirror.tar.gz 
oc-mirror

<first env>$ chmod +x oc-mirror
<first env>$ sudo cp oc-mirror /usr/local/bin/
```
Get a pull secret from from [Red Hat OpenShift Cluster Manager](https://console.redhat.com/openshift/install/pull-secret).

Copy the pull secret to the first environment and make a copy of it in JSON format:
```
$ scp pull-secret.txt lab-user@bastion-6lzvs.6lzvs.dynamic.opentlc.com:

<first env>$ cat pull-secret.txt | jq . > pull-secret-json.txt
```
Generate the base64-encoded username and password token for your mirror registry:
```
$ echo -n 'init:j4HPx6Gu18O52I70zlknDQw9do3thqKF'| base64 -w0
aW5pdDpqNEhQeDZHdTE4TzUySTcwemxrbkRRdzlkbzN0aHFLRg==
```

Edit the file with the pull secret in JSON format and add a section for the mirror registry.  If the port where the mirror registry is listening on is not 443, make sure to add the actual port number after the mirror registry hostname, for example "**bastion-sjmzk.sjmzk.dynamic.opentlc.com:8443**":
```
{
  "auths": {
   "cloud.openshift.com": {
    	"auth": "b3BlbnNoaWZ0LXJlbGVhc2...",
    	"email": "myemail@example.com"
   },
   "quay.io": {
      "auth": "b3BlbnNoaWZ0LXJlbGVhc...",
      "email": "myemail@example.com"
   },
   "registry.connect.redhat.com": {
      "auth": "NTIzMjU0MDB8dWhjLTFIaW...",
      "email": "myemail@example.com"
   },
   "registry.redhat.io": {
      "auth": "NTIzMjU0MDB8dWhjLTFIaWRhWDBvbHZ0Wnl...",
      "email": "myemail@example.com"
   },
    "bastion-sjmzk.sjmzk.dynamic.opentlc.com": {
      "auth": "aW5pdDpwa1p...EpRTTBsWDRvNQ==",
      "email": "myemail@example.com"
	  }
  }
}
```
Verify the JSON formating of the pull secret and install it so that it can be used by the oc CLI.
```
<first env>$ jq . pull-secret-json.txt
<first env>$ mkdir ~/.docker
<first env>$ cp pull-secret-json.txt ~/.docker/config.json
```
Verify the pull secret and create the image set configuration file template.  This command does not change or add anything to the mirror registry but verifies that the Red Hat public image registry and the local mirror registry are accessible, and creates a configuration template.

The **--registry** parameter uses the DNS name of the registry service, the port where the service listens on if this is different from  443, and a workspace or repository in the registry to hold the registry metadata (mirror/oc-mirror-metadata), the first time this command is executed the repository is created.

The “oc mirror” command can take a couple minutes to complete:
```
$ oc mirror init --registry bastion-vhnwk.vhnwk.dynamic.opentlc.com/mirror/oc-mirror-metadata > imageset-config.yaml
```

Edit the **imageset-config.yaml** file and include the required content to be mirrored to the local registry.  Examples and further details can be found in the [documentation](https://docs.openshift.com/container-platform/4.15/installing/disconnected_install/installing-mirroring-disconnected.html#oc-mirror-image-set-examples_installing-mirroring-disconnected)

To mirror a particular version of Openshift use a configuration like the following.  This configuration file only mirrors one particular OCP version for multiple architectures, but does not include any operators. This repository contains reference image configuration files at the **mirror/** directory.
```
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
storageConfig:
  registry:
    imageURL: bastion-vhnwk.vhnwk.dynamic.opentlc.com/mirroroc-mirror-metadata
    skipTLS: true
mirror:
  platform:
    architectures:
      - "multi"
      - "amd64"
    channels:
    - name: stable-4.15
      minVersion: 4.15.0
      maxVersion: 4.15.0
      type: ocp
```

Run the **oc mirror** command to get the container images for the requested OCP versions from the Red Hat public registries and store them in the local mirror registry.
```
$ oc mirror --config=./imageset-config.yaml docker://bastion-vhnwk.vhnwk.dynamic.opentlc.com/mirror/oc-mirror-metadata --dest-skip-tls
Checking push permissions for bastion-vhnwk.vhnwk.dynamic.opentlc.com
Found: oc-mirror-workspace/src/publish
Found: oc-mirror-workspace/src/v2
Found: oc-mirror-workspace/src/charts
Found: oc-mirror-workspace/src/release-signatures
No metadata detected, creating new workspace
...
info: Mirroring completed in 22m26.98s (49.56MB/s)
Writing image mapping to oc-mirror-workspace/results-1711562649/mapping.txt
Writing ICSP manifests to oc-mirror-workspace/results-1711562649
```
The mirroring command creates the directory **oc-mirror-workspace** in the same host where the command is being run, in the subdirectory **results-xxxx** can be found an imageContentSourcePolicy yaml definition that can be applied to an existing cluster or whose contents can be used to install a new cluster.  The image content source policy defines a mapping between the original source of the container images and the actual place to fetch them from.

```
$ cat oc-mirror-workspace/results-1711798302/imageContentSourcePolicy.yaml 
---
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: release-0
spec:
  repositoryDigestMirrors:
  - mirrors:
    - bastion-gwsmk.gwsmk.dynamic.opentlc.com/mirror/oc-mirror-metadata/openshift/release
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
  - mirrors:
    - bastion-gwsmk.gwsmk.dynamic.opentlc.com/mirror/oc-mirror-metadata/openshift/release-images
    source: quay.io/openshift-release-dev/ocp-release
```

Go to the quay web site to verify that the images are there:
![Quay website with Content](images/image30.png)


### Mirroring Operators

Additional information about this topic can be found in the knowledge base article [How to use the oc-mirror plug-in to mirror operators](https://access.redhat.com/solutions/6994677)
To include operator images in the mirror registry we need to collect some additional information.

The commands shown here take a few minutes to execute. Faster alternatives may exist using opm or grpcurl.

Operators are grouped into catalogs, to get the catalog list use the following command:
```
$ oc mirror list operators --catalogs --version=4.15
Available OpenShift OperatorHub catalogs:
OpenShift 4.15:
registry.redhat.io/redhat/redhat-operator-index:v4.15
registry.redhat.io/redhat/certified-operator-index:v4.15
registry.redhat.io/redhat/community-operator-index:v4.15
registry.redhat.io/redhat/redhat-marketplace-index:v4.15
```
Operators are stored as packages in the catalogs, find the available packages in each catalog and save the results:
```
$ oc mirror list operators --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.15 > redhat-catalog-packages-4.15.txt
$ oc mirror list operators --catalog=registry.redhat.io/redhat/certified-operator-index:v4.15 >certified-catalog-packages-4.15.txt
$ oc mirror list operators --catalog=registry.redhat.io/redhat/community-operator-index:v4.15 >community-catalog-packages-4.15.txt
$ oc mirror list operators --catalog=registry.redhat.io/redhat/redhat-marketplace-index:v4.15 >market-catalog-packages-4.15.txt
```
Find the operator that you want to install in the saved files, for example RHACM:
```
NAME                                          DISPLAY NAME                                             DEFAULT CHANNEL
3scale-operator                               Red Hat Integration - 3scale                             threescale-2.14
advanced-cluster-management                   Advanced Cluster Management for Kubernetes               release-2.10
amq-broker-rhel8                              Red Hat Integration - AMQ Broker for RHEL 8 (Multiarch)  7.11.x
...
```
Each operator can have different channels and several versions per channel.

Find the available channels for the selected operator.  Take note of the **default channel**, because _it must always be included in the image set configuration file_, even if the version that is going to be used is not in that channel.

This command is very slow, if you are requesting information about several operators, it can take a long time to complete.
```
$ oc mirror list operators --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.15 --package=advanced-cluster-management
NAME                         DISPLAY NAME                                DEFAULT CHANNEL
advanced-cluster-management  Advanced Cluster Management for Kubernetes  release-2.10

PACKAGE                      CHANNEL       HEAD
advanced-cluster-management  release-2.10  advanced-cluster-management.v2.10.0
advanced-cluster-management  release-2.9   advanced-cluster-management.v2.9.3
```
The command output shows that there are two channels available to install the RHACM operator: release-2.9 and release-2.10.

To find the package versions available within a particular channel use the command:
```
$ oc mirror list operators --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.15 --package=advanced-cluster-management --channel=release-2.9
VERSIONS
2.9.0
2.9.1
2.9.2
2.9.3
```
With the operator name, catalog, channel and versions, add a new section for the operators to the ImageSetConfiguration file.  Add the default channel for all the operators included in the configuration:
```
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
storageConfig:
  registry:
    imageURL: bastion-vhnwk.vhnwk.dynamic.opentlc.com/mirror/oc-mirror-metadata
    skipTLS: true
mirror:
  platform:
    architectures:
      - "multi"
      - "amd64"
    channels:
    - name: stable-4.15
      minVersion: 4.15.0
      maxVersion: 4.15.0
      type: ocp
  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.15
      packages:
      - name: advanced-cluster-management
        channels:
        - name: release-2.9
          minVersion: '2.9.2'
          maxVersion: '2.9.3'
        - name: release-2.10
          minVersion: '2.10.0'
          maxVersion: '2.10.0'
      - name: multicluster-engine
        channels:
        - name: stable-2.5
          minVersion: '2.5.1'
          maxVersion: '2.5.1'
```
In this example the packages are downloaded to the local filesystem (/var/mirror-registry/operator_catalog) instead of uploading them directly to the mirror registry.

Make sure that the local directory has enough space to hold all the images downloaded from the public Red Hat registries.

The directory is created if it does not exist, but the user running the command needs permissions to create that directory. If not then create the directory manually:
```
<first env>$ sudo mkdir /var/mirror-registry/operator-catalog
<first env>$ sudo chown lab-user: /var/mirror-registry/operator-catalog
<first env>$ ls -ld /var/mirror-registry/operator-catalog
drwxr-xr-x. 2 lab-user users 6 Mar 28 06:34 /var/mirror-registry/operator-catalog
```
Get the images:
```
<first env>$ oc mirror --config=./imageset-config-oper.yaml file:///var/mirror-registry/operator-catalog  --dest-skip-tls
Creating directory: /var/mirror-registry/operator-catalog/oc-mirror-workspace/src/publish
Creating directory: /var/mirror-registry/operator-catalog/oc-mirror-workspace/src/v2
Creating directory: /var/mirror-registry/operator-catalog/oc-mirror-workspace/src/charts
Creating directory: /var/mirror-registry/operator-catalog/oc-mirror-workspace/src/release-signatures
No metadata detected, creating new workspace
...
info: Mirroring completed in 11m42.86s (143.5MB/s)
Creating archive /var/mirror-registry/operator-catalog/mirror_seq1_000000.tar
```
This command takes quite some time to complete.

One reason it takes so long is because it compares the contents of the mirror registry with the packages to be downloaded so that it only downloads the packages that are not already in the mirror registry.  The command also removes any packages from the mirror registry that are not referenced in the imagesetconfiguration.

The obteined tar file needs to be uploaded to the mirror registry from the first bastion where the public DNS name can be used to access the mirror registry in the second environment, the tar file will not be copied to the second bastion and then loaded from there.

Load the file from the first bastion into the second bastion
```
$ oc mirror --from=/var/mirror-registry/operator-catalog/mirror_seq1_000000.tar docker://bastion-vhnwk.vhnwk.dynamic.opentlc.com  --dest-skip-tls
Checking push permissions for bastion-vhnwk.vhnwk.dynamic.opentlc.com
Publishing image set from archive "/var/mirror-registry/operator-catalog/mirror_seq1_000000.tar" to registry "bastion-vhnwk.vhnwk.dynamic.opentlc.com"
bastion-vhnwk.vhnwk.dynamic.opentlc.com/
  openshift/release
    manifests:
      sha256:2c6fd1f4b4fdbfbe34117e0477ad03b95792bb1eafa77f22608c107d7e2b8f19 -> 4.15.0-x86_64-alibaba-cloud-csi-driver
  stats: shared=0 unique=0 size=0B

phase 0:
  bastion-vhnwk.vhnwk.dynamic.opentlc.com openshift/release blobs=0 mounts=0 manifests=1 shared=0
....

```
When the commands finishes the tar file in the first environment can be deleted to save space.
```

```



### Updating the Images

The **oc mirror* command computes the difference between the images already present in the local mirror registry and the images requested in the image set configuration file.  Any images requested but not already present are downloaded and added to the local registry, and any images present in the local registry but not explicitly requested in the image set configuration file are removed from the local registry, therefore the **oc mirror* command can be used both to add new images to the local registry and the delete images that are not needed anymore.

Running the **oc mirror** command a second time without changing the image set configuration results in no images downloaded or deleted:
```
$ oc mirror --config=./imageset-config.yaml docker://bastion-vhnwk.vhnwk.dynamic.opentlc.com  --dest-skip-tls
Checking push permissions for bastion-vhnwk.vhnwk.dynamic.opentlc.com
Creating directory: oc-mirror-workspace/src/publish
Creating directory: oc-mirror-workspace/src/v2
Creating directory: oc-mirror-workspace/src/charts
Creating directory: oc-mirror-workspace/src/release-signatures
No new images detected, process stopping
```



If this is a disconnected installation, follow the instructions in section Mirror Registry to install the local registry and mirror the installation images.

