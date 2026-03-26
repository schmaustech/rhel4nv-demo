# RHEL4NV Demo 

**Goal**: The goal of this document is to provide the concise steps to enabling an updated kernel on an OpenShift deployment and then enabling the NVIDIA GPU Operator to compile and install properly.

## Environment

This procedure was tested on a 3 node hyperconverged 4.21.5 OpenShift cluster consisting of GB200 nodes.

The current kernels are listed as:

~~~bash
$ oc get nodes -o custom-columns=NAME:.metadata.name,KERNEL:.status.nodeInfo.kernelVersion
NAME                               KERNEL
erife-arm-openshift-4-21-5-gpu01   5.14.0-570.96.1.el9_6.aarch64
erife-arm-openshift-4-21-5-gpu02   5.14.0-570.96.1.el9_6.aarch64
erife-arm-openshift-4-21-5-gpu03   5.14.0-570.96.1.el9_6.aarch64
~~~~

However we want to get them to a RHEL10 state and so the process begins.

## Kernel Machine Configuration

To apply the custom kernel image we will use a machineconfig with the appropriate image defined in it.   As of this writing we are using this current image `quay.io/openshift-release-dev/ocp-v4.0-art-dev:4.21-10.1-ocp4nv-preview-202603182257-node-image`.   To begin we need to generate the machineconfig custom resource file and designate the node role and image inside.  Since this test cluster is a hyperconverged we need to specify the role as `master`.   For clusters with worker nodes seperated from control plane the role should be switched to `worker`.   Let's generate the following file.

~~~bash
$ cat <<EOF >99-kernel-machineconfig.yaml 
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 99-master-custom-os
spec:
  osImageURL: quay.io/openshift-release-dev/ocp-v4.0-art-dev:4.21-10.1-ocp4nv-preview-202603182257-node-image
EOF
~~~

Now we can create the machineconfig on the cluster.

~~~bash
$ oc create -f 99-kernel-machineconfig.yaml 
machineconfig.machineconfiguration.openshift.io/99-master-custom-os created
~~~

The application of the machineconfig will cause the nodes to get cordoned, drain and reboot one by one to make the change take effect.  We can observe this by using the following two commands to check the machineconfig pool state and also the node state.   Once the machineconfig pool is no longer in an updating state we know the process is complete.

~~~bash
$ oc get mcp
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-2cc4583126287d7d9537ea4032762592   False     True       False      3              0                   0                     0                      2d19h
worker   rendered-worker-6988cbbd70beeb96a8ffd32a5ba75f68   True      False      False      0              0                   0                     0                      2d19h

$ oc get nodes
NAME                               STATUS                     ROLES                         AGE     VERSION
erife-arm-openshift-4-21-5-gpu01   Ready,SchedulingDisabled   control-plane,master,worker   2d19h   v1.34.4
erife-arm-openshift-4-21-5-gpu02   Ready                      control-plane,master,worker   2d19h   v1.34.4
erife-arm-openshift-4-21-5-gpu03   Ready                      control-plane,master,worker   2d18h   v1.34.4
~~~

We can see as time progress node gpu01 was updated because version is different and now node gpu03 is in the process of reboot.

~~~bash
$ oc get nodes
NAME                               STATUS                        ROLES                         AGE     VERSION
erife-arm-openshift-4-21-5-gpu01   Ready                         control-plane,master,worker   2d19h   v1.34.2
erife-arm-openshift-4-21-5-gpu02   Ready                         control-plane,master,worker   2d19h   v1.34.4
erife-arm-openshift-4-21-5-gpu03   NotReady,SchedulingDisabled   control-plane,master,worker   2d19h   v1.34.4
~~~

Once the machineconfig has been applied we can confirm by looking on the nodes and checking the kernel version.  The results should look similar to the below.

~~~bash
$ oc get nodes -o custom-columns=NAME:.metadata.name,KERNEL:.status.nodeInfo.kernelVersion
NAME                               KERNEL
erife-arm-openshift-4-21-5-gpu01   6.12.0-211.4.el10nv.aarch64+64k
erife-arm-openshift-4-21-5-gpu02   6.12.0-211.4.el10nv.aarch64+64k
erife-arm-openshift-4-21-5-gpu03   6.12.0-211.4.el10nv.aarch64+64k
~~~

Now we can proceed to ensuring we have the right DTK image tagged.  Since we changed the kernel of our OpenShift environment we need to also point to a DTK image that is updated with our kernel version.   While you can build an updated DTK image I am using a pre-built one `quay.io/ravanelli/staging:dtk-4nv-03242026-211.4el0nv` known to work with this kernel version.   To ensure the GPU operator will pick it up we will run the following command to setup the tag.

~~~bash
$ oc tag -n openshift quay.io/ravanelli/staging:dtk-4nv-03242026-211.4el0nv openshift/driver-toolkit:10.1.20260126-0
Tag driver-toolkit:10.1.20260126-0 set to quay.io/ravanelli/staging:dtk-4nv-03242026-211.4el0nv.
~~~

We can validate the tag by the following.

~~~bash
$ oc get imagetag -n openshift|grep driver-toolkit
driver-toolkit:10.1.20260126-0               Tag         image/sha256:7dba910fd3a88a98e10b72f807b3c49bdb6dfd49f2cc9822846ce97b1cffdd06   1         4 seconds ago
driver-toolkit:9.6.20260303-1                Scheduled   image/sha256:20452648a60497336982393ea70789990072c363b752d56f5ab332acd1999b03   1         2 days ago
driver-toolkit:latest                        Scheduled   image/sha256:20452648a60497336982393ea70789990072c363b752d56f5ab332acd1999b03   3         46 hours ago
~~~

Next we need to install the NFD Operator and configure its instance to ensure our nodes are labeled properly for the NVIDIA GPU Operator.  First let's generate the operator customer resource file to install the NFD Operator.

~~~bash
$ cat <<EOF >nfd-operator.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-nfd
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-nfd
  namespace: openshift-nfd
spec:
  targetNamespaces:
    - openshift-nfd
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: nfd
  namespace: openshift-nfd
spec:
  channel: "stable"
  installPlanApproval: Automatic
  name: nfd
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
~~~

Then we can create the resouce on the cluster.

~~~bash
$ oc create -f nfd-operator.yaml 
namespace/openshift-nfd created
operatorgroup.operators.coreos.com/openshift-nfd created
subscription.operators.coreos.com/nfd created
~~~

Next validate the NFD operator is running.

~~~bash
$ oc get pods -n openshift-nfd
NAME                                     READY   STATUS    RESTARTS   AGE
nfd-controller-manager-f945d98dd-nrp5w   1/1     Running   0          30s
~~~

Now we are ready to generate the NFD instance custom resource file.

~~~bash
$ cat <<EOF >nfd-instance.yaml 
apiVersion: nfd.openshift.io/v1
kind: NodeFeatureDiscovery
metadata:
  name: nfd-instance
  namespace: openshift-nfd
spec:
  instance: ''
  operand:
    servicePort: 12000
  prunerOnDelete: false
  topologyUpdater: false
  workerConfig:
    configData: |
      core:
        sleepInterval: 60s
      sources:
        pci:
          deviceClassWhitelist:
            - "02"
            - "03"
            - "0200"
            - "0207"
            - "12"
          deviceLabelFields:
            - "vendor"
EOF
~~~

With the file ready we can create it on the cluster.

~~~bash
$ oc create -f nfd-instance.yaml
nodefeaturediscovery.nfd.openshift.io/nfd-instance created
~~~

We can validate the NFD Operator is ready by checking the pod states and checking labels.

~~~bash
$ oc create -f nfd-operator.yaml 
namespace/openshift-nfd created
operatorgroup.operators.coreos.com/openshift-nfd created
subscription.operators.coreos.com/nfd created

$ oc get nodes --show-labels | grep feature.node
~~~

If everything is still looking good we can now move onto installing the NVIDIA GPU Operator.  The first step here is to generate the NVIDIA Network Operator custom resource file.

~~~bash
$ cat <<EOF >nvidia-gpu-operator.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: nvidia-gpu-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: nvidia-gpu-operator
  namespace: nvidia-gpu-operator
spec:
  targetNamespaces:
    - nvidia-gpu-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: nvidia-gpu-operator
  namespace: nvidia-gpu-operator
spec:
  channel: v26.3
  installPlanApproval: Automatic
  name: gpu-operator-certified
  source: certified-operators
  sourceNamespace: openshift-marketplace
EOF
~~~

Now we can create the NVIDIA Operator on the cluster.

~~~bash
$ oc create -f nvidia-gpu-operator.yaml
~~~

We can validate the operator is running with the following.

~~~bash
$ oc get pods -n nvidia-gpu-operator
~~~

Finally we are ready to deploy our gpu cluster policy.   The following is an example cluster policy we used in this environment.  I should point out at the time of this writing we required a special build driver container from NVIDIA as the default one that came with v26.3 of the GPU Operator did not work for us.   This is why we see the repository as `quay.io/redhat_emp1/ecosys-nvidia` for the driver because that is where we staged our driver. 

~~~bash
$ cat <<EOF >gpu-clusterpolicy.yaml 
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: gpu-cluster-policy
spec:
  vgpuDeviceManager:
    config:
      default: default
    enabled: true
  migManager:
    config:
      default: all-disabled
    enabled: true
  operator:
    runtimeClass: nvidia
    use_ocp_driver_toolkit: true
  dcgm:
    enabled: true
  gfd:
    enabled: true
  dcgmExporter:
    config:
      name: ''
    serviceMonitor:
      enabled: true
    enabled: true
  cdi:
    default: false
    enabled: false
    nriPluginEnabled: false
  driver:
    licensingConfig:
      nlsEnabled: true
      secretName: ''
    kernelModuleType: open
    certConfig:
      name: ''
    kernelModuleConfig:
      name: ''
    upgradePolicy:
      autoUpgrade: true
      drain:
        deleteEmptyDir: false
        enable: false
        force: false
        timeoutSeconds: 300
      maxParallelUpgrades: 1
      maxUnavailable: 25%
      podDeletion:
        deleteEmptyDir: false
        force: false
        timeoutSeconds: 300
      waitForCompletion:
        timeoutSeconds: 0
    repoConfig:
      configMapName: ''
    virtualTopology:
      config: ''
    enabled: true
    useNvidiaDriverCRD: false
    repository: quay.io/redhat_emp1/ecosys-nvidia
    image: driver
    version: 580.126.09
  devicePlugin:
    config:
      name: ''
      default: ''
    mps:
      root: /run/nvidia/mps
    enabled: true
  gdrcopy:
    enabled: true
  kataManager:
    config:
      artifactsDir: /opt/nvidia-gpu-operator/artifacts/runtimeclasses
  mig:
    strategy: single
  kataSandboxDevicePlugin:
    enabled: true
  ccManager:
    enabled: true
  sandboxDevicePlugin:
    enabled: true
  validator:
    plugin:
      env: []
  nodeStatusExporter:
    enabled: true
  daemonsets:
    rollingUpdate:
      maxUnavailable: '1'
    updateStrategy: RollingUpdate
  sandboxWorkloads:
    defaultWorkload: container
    mode: kubevirt
    enabled: false
  gds:
    enabled: false
  vgpuManager:
    enabled: false
  vfioManager:
    enabled: true
  toolkit:
    installDir: /usr/local/nvidia
    enabled: true
EOF
~~~
