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

Ssh into the bastion node and verify the DNS records for the cluster.  The IPs associated with the API and wildcard \*.apps are public, but they are NATed to internal private IPs in the environment, this is specified in the email received from RHDP:

```
$ dig +short api.glnm2.dynamic.opentlc.com
3.223.59.140
$ dig +short *.apps.glnm2.dynamic.opentlc.com
34.198.235.139
```
Get the installer, oc client and pull secret from [the Red Hat Console](https://console.redhat.com/openshift/install) and copy them to the bastion host. 

Uncompress the tar files and put them in the running path:

```
$ scp ~/Descargas/openshift-* /home/jjerezro/Descargas/pull-secret.txt   \
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

This section describes how to deploy an Openshift cluster on a disconnected environment, considering the special limitations of the [demo.redhat.com](https://demo.redhat.com).

The disconnected environment in this example does not have direct access to the public Red Hat image registries in order to fetch the images required for the installation.  A local registry must be provisioned so the cluster nodes can download the images from it and the installation can succeed. 

The local registry used is the [quay mirror registry](https://github.com/quay/mirror-registry)

The mirror registry service name needs to be resolvable by DNS, by any client accessing it.  Considering demo.redhat.com limitations regarding DNS in vSphere based environments, in which it is not possible to create new or modify existing DNS records, we are going to deploy the mirror registry remotely from one VMware Cloud Open Environment onto another so the bastion host on the second environment can be used as the mirror registry for the first or any other environment.  The bastion host public DNS name in the second environment will be used to access the mirror registry.

![Mirror Registry Across Environments](images/MirrorRegistryAcross.drawio.png)






If this is a disconnected installation, follow the instructions in section Mirror Registry to install the local registry and mirror the installation images.

