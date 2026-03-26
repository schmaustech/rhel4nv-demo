# RHEL4NV Demo 

**Goal**: The goal of this document is to provide the concise steps to enabling an updated kernel on an OpenShift deployment and then enabling the NVIDIA Network Operator and NVIDIA GPU Operator to compile and install properly.

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

However we want to get them to a RHEL4NV state and so the process begins.


## Workflow Sections

- [Kernel Machine Configuration](#kernel-machine-configuration)
- [Driver ToolKit Image](#driver-toolkit-image)
- [NFD Operator](#nfd-operator)
- [NVIDIA Network Operator](#nvidia-network-operator)
- [NVIDIA GPU Operator](#nvidia-gpu-operator)
- [OpenShift Virtualization](#openshift-virtualization)

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

## Driver ToolKit Image 

Now we can proceed to ensuring we have the right DTK image tagged.  Since we changed the kernel of our OpenShift environment we need to also point to a DTK image that is updated with our kernel version.   While you can build an updated DTK image I am using a pre-built one `quay.io/ravanelli/staging:dtk-4nv-03262026-211.4el0nv` known to work with this kernel version.   To ensure the GPU operator will pick it up we will run the following command to setup the tag.

~~~bash
$ oc tag -n quay.io/ravanelli/staging:dtk-4nv-03262026-211.4el0nv openshift/driver-toolkit:10.1.20260126-0
Tag driver-toolkit:10.1.20260126-0 set to quay.io/ravanelli/staging:dtk-4nv-03262026-211.4el0nv.
~~~

We can validate the tag by the following.

~~~bash
$ oc get imagetag -n openshift|grep driver-toolkit
driver-toolkit:10.1.20260126-0               Tag         image/sha256:8f426c63d003b6e0fbec319f9615f6da743de2b6e415b90bb55f007c30cd0d25   2         3 hours ago
driver-toolkit:9.6.20260303-1                Scheduled   image/sha256:20452648a60497336982393ea70789990072c363b752d56f5ab332acd1999b03   1         3 days ago
driver-toolkit:latest                        Scheduled   image/sha256:20452648a60497336982393ea70789990072c363b752d56f5ab332acd1999b03   3         2 days ago
~~~

## NFD Operator 

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
erife-arm-openshift-4-21-5-gpu01   Ready    control-plane,master,worker   2d21h   v1.34.2   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,cpumanager=false,feature.node.kubernetes.io/cpu-cpuid.AES=true,feature.node.kubernetes.io/cpu-cpuid.ASIMD=true,feature.node.kubernetes.io/cpu-cpuid.ASIMDDP=true,feature.node.kubernetes.io/cpu-cpuid.ASIMDFHM=true,feature.node.kubernetes.io/cpu-cpuid.ASIMDHP=true,feature.node.kubernetes.io/cpu-cpuid.ASIMDRDM=true,feature.node.kubernetes.io/cpu-cpuid.ATOMICS=true,feature.node.kubernetes.io/cpu-cpuid.BF16=true,feature.node.kubernetes.io/cpu-cpuid.BTI=true,feature.node.kubernetes.io/cpu-cpuid.CPUID=true,feature.node.kubernetes.io/cpu-cpuid.CRC32=true,feature.node.kubernetes.io/cpu-cpuid.DCPODP=true,feature.node.kubernetes.io/cpu-cpuid.DCPOP=true,feature.node.kubernetes.io/cpu-cpuid.DGH=true,feature.node.kubernetes.io/cpu-cpuid.DIT=true,feature.node.kubernetes.io/cpu-cpuid.EVTSTRM=true,feature.node.kubernetes.io/cpu-cpuid.FCMA=true,feature.node.kubernetes.io/cpu-cpuid.FLAGM2=true,feature.node.kubernetes.io/cpu-cpuid.FLAGM=true,feature.node.kubernetes.io/cpu-cpuid.FP=true,feature.node.kubernetes.io/cpu-cpuid.FPHP=true,feature.node.kubernetes.io/cpu-cpuid.FRINT=true,feature.node.kubernetes.io/cpu-cpuid.I8MM=true,feature.node.kubernetes.io/cpu-cpuid.ILRCPC=true,feature.node.kubernetes.io/cpu-cpuid.JSCVT=true,feature.node.kubernetes.io/cpu-cpuid.LRCPC=true,feature.node.kubernetes.io/cpu-cpuid.PACA=true,feature.node.kubernetes.io/cpu-cpuid.PACG=true,feature.node.kubernetes.io/cpu-cpuid.PMULL=true,feature.node.kubernetes.io/cpu-cpuid.SB=true,feature.node.kubernetes.io/cpu-cpuid.SHA1=true,feature.node.kubernetes.io/cpu-cpuid.SHA2=true,feature.node.kubernetes.io/cpu-cpuid.SHA3=true,feature.node.kubernetes.io/cpu-cpuid.SHA512=true,feature.node.kubernetes.io/cpu-cpuid.SM3=true,feature.node.kubernetes.io/cpu-cpuid.SM4=true,feature.node.kubernetes.io/cpu-cpuid.SVE2=true,feature.node.kubernetes.io/cpu-cpuid.SVE=true,feature.node.kubernetes.io/cpu-cpuid.SVEAES=true,feature.node.kubernetes.io/cpu-cpuid.SVEBF16=true,feature.node.kubernetes.io/cpu-cpuid.SVEBITPERM=true,feature.node.kubernetes.io/cpu-cpuid.SVEI8MM=true,feature.node.kubernetes.io/cpu-cpuid.SVEPMULL=true,feature.node.kubernetes.io/cpu-cpuid.SVESHA3=true,feature.node.kubernetes.io/cpu-cpuid.SVESM4=true,feature.node.kubernetes.io/cpu-cpuid.USCAT=true,feature.node.kubernetes.io/cpu-hardware_multithreading=false,feature.node.kubernetes.io/cpu-model.family=15,feature.node.kubernetes.io/cpu-model.id=54512,feature.node.kubernetes.io/cpu-model.vendor_id=ARM,feature.node.kubernetes.io/kernel-config.NO_HZ=true,feature.node.kubernetes.io/kernel-config.NO_HZ_FULL=true,feature.node.kubernetes.io/kernel-selinux.enabled=true,feature.node.kubernetes.io/kernel-version.full=6.12.0-211.4.el10nv.aarch64_64k,feature.node.kubernetes.io/kernel-version.major=6,feature.node.kubernetes.io/kernel-version.minor=12,feature.node.kubernetes.io/kernel-version.revision=0,feature.node.kubernetes.io/memory-numa=true,feature.node.kubernetes.io/network-sriov.capable=true,feature.node.kubernetes.io/pci-10de.present=true,feature.node.kubernetes.io/pci-10de.sriov.capable=true,feature.node.kubernetes.io/pci-15b3.present=true,feature.node.kubernetes.io/pci-15b3.sriov.capable=true,feature.node.kubernetes.io/pci-1a03.present=true,feature.node.kubernetes.io/pci-8086.present=true,feature.node.kubernetes.io/rdma.capable=true,feature.node.kubernetes.io/storage-nonrotationaldisk=true,feature.node.kubernetes.io/system-os_release.ID=rhel,feature.node.kubernetes.io/system-os_release.OPENSHIFT_VERSION=4.21,feature.node.kubernetes.io/system-os_release.OSTREE_VERSION=10.1.20260126-0,feature.node.kubernetes.io/system-os_release.VERSION_ID.major=10,feature.node.kubernetes.io/system-os_release.VERSION_ID.minor=1,feature.node.kubernetes.io/system-os_release.VERSION_ID=10.1,kubernetes.io/arch=arm64,kubernetes.io/hostname=erife-arm-openshift-4-21-5-gpu01,kubernetes.io/os=linux,machine-type.node.kubevirt.io/virt-rhel9.0.0=true,machine-type.node.kubevirt.io/virt-rhel9.2.0=true,machine-type.node.kubevirt.io/virt-rhel9.4.0=true,machine-type.node.kubevirt.io/virt-rhel9.6.0=true,machine-type.node.kubevirt.io/virt=true,network.nvidia.com/operator.mofed.wait=false,network.nvidia.com/operator.nic-configuration.wait=false,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node-role.kubernetes.io/worker=,node.openshift.io/os_id=rhel,nvidia.com/gpu-driver-upgrade-state=upgrade-done,nvidia.com/gpu.deploy.container-toolkit=true,nvidia.com/gpu.deploy.dcgm-exporter=true,nvidia.com/gpu.deploy.dcgm=true,nvidia.com/gpu.deploy.device-plugin=true,nvidia.com/gpu.deploy.driver=true,nvidia.com/gpu.deploy.gpu-feature-discovery=true,nvidia.com/gpu.deploy.node-status-exporter=true,nvidia.com/gpu.deploy.nvsm=,nvidia.com/gpu.deploy.operator-validator=true,nvidia.com/gpu.present=true,topology.topolvm.io/node=erife-arm-openshift-4-21-5-gpu01
erife-arm-openshift-4-21-5-gpu02   Ready    control-plane,master,worker   2d21h   v1.34.2   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,cpumanager=false,feature.node.kubernetes.io/cpu-cpuid.AES=true,feature.node.kubernetes.io/cpu-cpuid.ASIMD=true,feature.node.kubernetes.io/cpu-cpuid.ASIMDDP=true,feature.node.kubernetes.io/cpu-cpuid.ASIMDFHM=true,feature.node.kubernetes.io/cpu-cpuid.ASIMDHP=true,feature.node.kubernetes.io/cpu-cpuid.ASIMDRDM=true,feature.node.kubernetes.io/cpu-cpuid.ATOMICS=true,feature.node.kubernetes.io/cpu-cpuid.BF16=true,feature.node.kubernetes.io/cpu-cpuid.BTI=true,feature.node.kubernetes.io/cpu-cpuid.CPUID=true,feature.node.kubernetes.io/cpu-cpuid.CRC32=true,feature.node.kubernetes.io/cpu-cpuid.DCPODP=true,feature.node.kubernetes.io/cpu-cpuid.DCPOP=true,feature.node.kubernetes.io/cpu-cpuid.DGH=true,feature.node.kubernetes.io/cpu-cpuid.DIT=true,feature.node.kubernetes.io/cpu-cpuid.EVTSTRM=true,feature.node.kubernetes.io/cpu-cpuid.FCMA=true,feature.node.kubernetes.io/cpu-cpuid.FLAGM2=true,feature.node.kubernetes.io/cpu-cpuid.FLAGM=true,feature.node.kubernetes.io/cpu-cpuid.FP=true,feature.node.kubernetes.io/cpu-cpuid.FPHP=true,feature.node.kubernetes.io/cpu-cpuid.FRINT=true,feature.node.kubernetes.io/cpu-cpuid.I8MM=true,feature.node.kubernetes.io/cpu-cpuid.ILRCPC=true,feature.node.kubernetes.io/cpu-cpuid.JSCVT=true,feature.node.kubernetes.io/cpu-cpuid.LRCPC=true,feature.node.kubernetes.io/cpu-cpuid.PACA=true,feature.node.kubernetes.io/cpu-cpuid.PACG=true,feature.node.kubernetes.io/cpu-cpuid.PMULL=true,feature.node.kubernetes.io/cpu-cpuid.SB=true,feature.node.kubernetes.io/cpu-cpuid.SHA1=true,feature.node.kubernetes.io/cpu-cpuid.SHA2=true,feature.node.kubernetes.io/cpu-cpuid.SHA3=true,feature.node.kubernetes.io/cpu-cpuid.SHA512=true,feature.node.kubernetes.io/cpu-cpuid.SM3=true,feature.node.kubernetes.io/cpu-cpuid.SM4=true,feature.node.kubernetes.io/cpu-cpuid.SVE2=true,feature.node.kubernetes.io/cpu-cpuid.SVE=true,feature.node.kubernetes.io/cpu-cpuid.SVEAES=true,feature.node.kubernetes.io/cpu-cpuid.SVEBF16=true,feature.node.kubernetes.io/cpu-cpuid.SVEBITPERM=true,feature.node.kubernetes.io/cpu-cpuid.SVEI8MM=true,feature.node.kubernetes.io/cpu-cpuid.SVEPMULL=true,feature.node.kubernetes.io/cpu-cpuid.SVESHA3=true,feature.node.kubernetes.io/cpu-cpuid.SVESM4=true,feature.node.kubernetes.io/cpu-cpuid.USCAT=true,feature.node.kubernetes.io/cpu-hardware_multithreading=false,feature.node.kubernetes.io/cpu-model.family=15,feature.node.kubernetes.io/cpu-model.id=54512,feature.node.kubernetes.io/cpu-model.vendor_id=ARM,feature.node.kubernetes.io/kernel-config.NO_HZ=true,feature.node.kubernetes.io/kernel-config.NO_HZ_FULL=true,feature.node.kubernetes.io/kernel-selinux.enabled=true,feature.node.kubernetes.io/kernel-version.full=6.12.0-211.4.el10nv.aarch64_64k,feature.node.kubernetes.io/kernel-version.major=6,feature.node.kubernetes.io/kernel-version.minor=12,feature.node.kubernetes.io/kernel-version.revision=0,feature.node.kubernetes.io/memory-numa=true,feature.node.kubernetes.io/network-sriov.capable=true,feature.node.kubernetes.io/pci-10de.present=true,feature.node.kubernetes.io/pci-10de.sriov.capable=true,feature.node.kubernetes.io/pci-15b3.present=true,feature.node.kubernetes.io/pci-15b3.sriov.capable=true,feature.node.kubernetes.io/pci-1a03.present=true,feature.node.kubernetes.io/pci-8086.present=true,feature.node.kubernetes.io/rdma.capable=true,feature.node.kubernetes.io/storage-nonrotationaldisk=true,feature.node.kubernetes.io/system-os_release.ID=rhel,feature.node.kubernetes.io/system-os_release.OPENSHIFT_VERSION=4.21,feature.node.kubernetes.io/system-os_release.OSTREE_VERSION=10.1.20260126-0,feature.node.kubernetes.io/system-os_release.VERSION_ID.major=10,feature.node.kubernetes.io/system-os_release.VERSION_ID.minor=1,feature.node.kubernetes.io/system-os_release.VERSION_ID=10.1,kubernetes.io/arch=arm64,kubernetes.io/hostname=erife-arm-openshift-4-21-5-gpu02,kubernetes.io/os=linux,machine-type.node.kubevirt.io/virt-rhel9.0.0=true,machine-type.node.kubevirt.io/virt-rhel9.2.0=true,machine-type.node.kubevirt.io/virt-rhel9.4.0=true,machine-type.node.kubevirt.io/virt-rhel9.6.0=true,machine-type.node.kubevirt.io/virt=true,network.nvidia.com/operator.mofed.wait=false,network.nvidia.com/operator.nic-configuration.wait=false,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node-role.kubernetes.io/worker=,node.openshift.io/os_id=rhel,nvidia.com/gpu-driver-upgrade-state=upgrade-done,nvidia.com/gpu.deploy.container-toolkit=true,nvidia.com/gpu.deploy.dcgm-exporter=true,nvidia.com/gpu.deploy.dcgm=true,nvidia.com/gpu.deploy.device-plugin=true,nvidia.com/gpu.deploy.driver=true,nvidia.com/gpu.deploy.gpu-feature-discovery=true,nvidia.com/gpu.deploy.node-status-exporter=true,nvidia.com/gpu.deploy.nvsm=,nvidia.com/gpu.deploy.operator-validator=true,nvidia.com/gpu.present=true,topology.topolvm.io/node=erife-arm-openshift-4-21-5-gpu02
erife-arm-openshift-4-21-5-gpu03   Ready    control-plane,master,worker   2d20h   v1.34.2   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,cpumanager=false,feature.node.kubernetes.io/cpu-cpuid.AES=true,feature.node.kubernetes.io/cpu-cpuid.ASIMD=true,feature.node.kubernetes.io/cpu-cpuid.ASIMDDP=true,feature.node.kubernetes.io/cpu-cpuid.ASIMDFHM=true,feature.node.kubernetes.io/cpu-cpuid.ASIMDHP=true,feature.node.kubernetes.io/cpu-cpuid.ASIMDRDM=true,feature.node.kubernetes.io/cpu-cpuid.ATOMICS=true,feature.node.kubernetes.io/cpu-cpuid.BF16=true,feature.node.kubernetes.io/cpu-cpuid.BTI=true,feature.node.kubernetes.io/cpu-cpuid.CPUID=true,feature.node.kubernetes.io/cpu-cpuid.CRC32=true,feature.node.kubernetes.io/cpu-cpuid.DCPODP=true,feature.node.kubernetes.io/cpu-cpuid.DCPOP=true,feature.node.kubernetes.io/cpu-cpuid.DGH=true,feature.node.kubernetes.io/cpu-cpuid.DIT=true,feature.node.kubernetes.io/cpu-cpuid.EVTSTRM=true,feature.node.kubernetes.io/cpu-cpuid.FCMA=true,feature.node.kubernetes.io/cpu-cpuid.FLAGM2=true,feature.node.kubernetes.io/cpu-cpuid.FLAGM=true,feature.node.kubernetes.io/cpu-cpuid.FP=true,feature.node.kubernetes.io/cpu-cpuid.FPHP=true,feature.node.kubernetes.io/cpu-cpuid.FRINT=true,feature.node.kubernetes.io/cpu-cpuid.I8MM=true,feature.node.kubernetes.io/cpu-cpuid.ILRCPC=true,feature.node.kubernetes.io/cpu-cpuid.JSCVT=true,feature.node.kubernetes.io/cpu-cpuid.LRCPC=true,feature.node.kubernetes.io/cpu-cpuid.PACA=true,feature.node.kubernetes.io/cpu-cpuid.PACG=true,feature.node.kubernetes.io/cpu-cpuid.PMULL=true,feature.node.kubernetes.io/cpu-cpuid.SB=true,feature.node.kubernetes.io/cpu-cpuid.SHA1=true,feature.node.kubernetes.io/cpu-cpuid.SHA2=true,feature.node.kubernetes.io/cpu-cpuid.SHA3=true,feature.node.kubernetes.io/cpu-cpuid.SHA512=true,feature.node.kubernetes.io/cpu-cpuid.SM3=true,feature.node.kubernetes.io/cpu-cpuid.SM4=true,feature.node.kubernetes.io/cpu-cpuid.SVE2=true,feature.node.kubernetes.io/cpu-cpuid.SVE=true,feature.node.kubernetes.io/cpu-cpuid.SVEAES=true,feature.node.kubernetes.io/cpu-cpuid.SVEBF16=true,feature.node.kubernetes.io/cpu-cpuid.SVEBITPERM=true,feature.node.kubernetes.io/cpu-cpuid.SVEI8MM=true,feature.node.kubernetes.io/cpu-cpuid.SVEPMULL=true,feature.node.kubernetes.io/cpu-cpuid.SVESHA3=true,feature.node.kubernetes.io/cpu-cpuid.SVESM4=true,feature.node.kubernetes.io/cpu-cpuid.USCAT=true,feature.node.kubernetes.io/cpu-hardware_multithreading=false,feature.node.kubernetes.io/cpu-model.family=15,feature.node.kubernetes.io/cpu-model.id=54512,feature.node.kubernetes.io/cpu-model.vendor_id=ARM,feature.node.kubernetes.io/kernel-config.NO_HZ=true,feature.node.kubernetes.io/kernel-config.NO_HZ_FULL=true,feature.node.kubernetes.io/kernel-selinux.enabled=true,feature.node.kubernetes.io/kernel-version.full=6.12.0-211.4.el10nv.aarch64_64k,feature.node.kubernetes.io/kernel-version.major=6,feature.node.kubernetes.io/kernel-version.minor=12,feature.node.kubernetes.io/kernel-version.revision=0,feature.node.kubernetes.io/memory-numa=true,feature.node.kubernetes.io/network-sriov.capable=true,feature.node.kubernetes.io/pci-10de.present=true,feature.node.kubernetes.io/pci-10de.sriov.capable=true,feature.node.kubernetes.io/pci-15b3.present=true,feature.node.kubernetes.io/pci-15b3.sriov.capable=true,feature.node.kubernetes.io/pci-1a03.present=true,feature.node.kubernetes.io/pci-8086.present=true,feature.node.kubernetes.io/rdma.capable=true,feature.node.kubernetes.io/storage-nonrotationaldisk=true,feature.node.kubernetes.io/system-os_release.ID=rhel,feature.node.kubernetes.io/system-os_release.OPENSHIFT_VERSION=4.21,feature.node.kubernetes.io/system-os_release.OSTREE_VERSION=10.1.20260126-0,feature.node.kubernetes.io/system-os_release.VERSION_ID.major=10,feature.node.kubernetes.io/system-os_release.VERSION_ID.minor=1,feature.node.kubernetes.io/system-os_release.VERSION_ID=10.1,kubernetes.io/arch=arm64,kubernetes.io/hostname=erife-arm-openshift-4-21-5-gpu03,kubernetes.io/os=linux,machine-type.node.kubevirt.io/virt-rhel9.0.0=true,machine-type.node.kubevirt.io/virt-rhel9.2.0=true,machine-type.node.kubevirt.io/virt-rhel9.4.0=true,machine-type.node.kubevirt.io/virt-rhel9.6.0=true,machine-type.node.kubevirt.io/virt=true,network.nvidia.com/operator.mofed.wait=false,network.nvidia.com/operator.nic-configuration.wait=false,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node-role.kubernetes.io/worker=,node.openshift.io/os_id=rhel,nvidia.com/gpu-driver-upgrade-state=upgrade-done,nvidia.com/gpu.deploy.container-toolkit=true,nvidia.com/gpu.deploy.dcgm-exporter=true,nvidia.com/gpu.deploy.dcgm=true,nvidia.com/gpu.deploy.device-plugin=true,nvidia.com/gpu.deploy.driver=true,nvidia.com/gpu.deploy.gpu-feature-discovery=true,nvidia.com/gpu.deploy.node-status-exporter=true,nvidia.com/gpu.deploy.nvsm=,nvidia.com/gpu.deploy.operator-validator=true,nvidia.com/gpu.present=true,topology.topolvm.io/node=erife-arm-openshift-4-21-5-gpu03
~~~

## NVIDIA Network Operator

With NFD installed we can move onto installing and configuring the NVIDIA Network Operator. The first thing we will need to do is generate the custom resource file to install the operator.  Note that as of this writing the v26.1 release of the NVIDIA Network Operator was not available for OpenShift 4.21 so v25.10 of the operator is being used here.

~~~bash
$ cat <<EOF >nno-operator.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: nvidia-network-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: nvidia-network-operator
  namespace: nvidia-network-operator
spec:
  targetNamespaces:
  - nvidia-network-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: nvidia-network-operator
  namespace: nvidia-network-operator
spec:
  channel: v25.10
  installPlanApproval: Automatic
  name: nvidia-network-operator
  source: certified-operators
  sourceNamespace: openshift-marketplace
EOF
~~~

With the file generated we can create it on the cluster.

~~~bash
$ oc create -f nno-operator.yaml 
namespace/nvidia-network-operator created
operatorgroup.operators.coreos.com/nvidia-network-operator created
subscription.operators.coreos.com/nvidia-network-operator created
~~~

We can vaidate the NVIDIA Network Operator is running by the following.

~~~bash
$ oc get pods -n nvidia-network-operator
NAME                                                         READY   STATUS        RESTARTS   AGE
nvidia-network-operator-controller-manager-87cf46bcc-h6wzn   1/1     Running       0          116m
~~~

Now that the operator is up and running we can go ahead and configure a basic driver policy. In order to match the kernel we have installed on this cluster I had to do some tagging magic from NVIDIA's registry and then push it up to our own private registry.

~~~bash
# podman tag nvcr.io/nvidia/mellanox/doca-driver:doca3.3.0-26.01-1.0.0.0-0-rhel10.0-arm64 quay.io/redhat_emp1/ecosys-nvidia/doca-driver:doca3.3.0-26.01-1.0.0.0-0-rhel10.1-arm64
# podman push quay.io/redhat_emp1/ecosys-nvidia/doca-driver:doca3.3.0-26.01-1.0.0.0-0-rhel10.1-arm64
~~~

Now in the NicClusterPolicy we can specific the DOCA version and it will be able to find the right tag from our registry.

~~~bash
$ cat <<EOF >nicclusterpolicy.yaml 
apiVersion: mellanox.com/v1alpha1
kind: NicClusterPolicy
metadata:
  name: nic-cluster-policy
spec:
  ofedDriver:
    imagePullSecrets: []
    readinessProbe:
      initialDelaySeconds: 10
      periodSeconds: 30
    forcePrecompiled: false
    terminationGracePeriodSeconds: 300
    repository: quay.io/redhat_emp1/ecosys-nvidia
    livenessProbe:
      initialDelaySeconds: 30
      periodSeconds: 30
    upgradePolicy:
      autoUpgrade: true
      drain:
        deleteEmptyDir: true
        enable: true
        force: true
        podSelector: ''
        timeoutSeconds: 300
      maxParallelUpgrades: 1
      safeLoad: false
    startupProbe:
      initialDelaySeconds: 10
      periodSeconds: 20
    version: doca3.3.0-26.01-1.0.0.0-0
    image: doca-driver
    env:
    - name: UNLOAD_STORAGE_MODULES
      value: "true"
    - name: RESTORE_DRIVER_ON_POD_TERMINATION
      value: "true"
    - name: CREATE_IFNAMES_UDEV
      value: "true"
    - name: ENTRYPOINT_DEBUG
      value: 'true'
EOF
~~~

Once we have generated the NicClusterPolicy we can create it on the cluster.

~~~bash
$ oc create -f nicclusterpolicy.yaml 
nicclusterpolicy.mellanox.com/nic-cluster-policy created
~~~

The NicClusterPolicy will take ~5 minutes to complete as it builds the drivers and then ultimately unloads the in-tree mlx5 drivers and loads the out-of-tree drivers.  After that time we should see mofed pods that are in a 2/2 ready state.  The number of mofed pods will be based on the number of nodes that have valid Mellanox devices in them.

~~~bash
$ oc get pods -n nvidia-network-operator
NAME                                                         READY   STATUS    RESTARTS        AGE
mofed-rhel10.1-547d75b4d8-ds-9psph                           2/2     Running   0               21m
mofed-rhel10.1-547d75b4d8-ds-j5mcf                           2/2     Running   0               21m
mofed-rhel10.1-547d75b4d8-ds-zx6fj                           2/2     Running   0               13m
nvidia-network-operator-controller-manager-87cf46bcc-h6wzn   1/1     Running   5 (6m27s ago)   5h44m
~~~

We can further validate by going into one of the mofed pods and running a few commands.

~~~bash
$ oc rsh -n nvidia-network-operator mofed-rhel10.1-547d75b4d8-ds-9psph
Defaulted container "mofed-container" out of: mofed-container, openshift-driver-toolkit-ctr, network-operator-init-container (init)

sh-5.2# lsmod|grep mlx5
mlx5_vdpa             196608  0
mlx5_ib               720896  0
ib_uverbs             393216  2 rdma_ucm,mlx5_ib
mlx5_core            3080192  1 mlx5_ib
mlxfw                 262144  1 mlx5_core
mlxdevm               589824  1 mlx5_core
ib_core               655360  8 rdma_cm,ib_ipoib,iw_cm,ib_umad,rdma_ucm,ib_uverbs,mlx5_ib,ib_cm
mlx_compat            196608  12 rdma_cm,ib_ipoib,mlxdevm,iw_cm,ib_umad,mlx5_vdpa,ib_core,rdma_ucm,ib_uverbs,mlx5_ib,ib_cm,mlx5_core
macsec                262144  1 mlx5_ib
psample               262144  2 openvswitch,mlx5_core
pci_hyperv_intf       196608  1 mlx5_core
tls                   327680  1 mlx5_core

sh-5.2# ofed_info 
OFED-internal-26.01-1.0.0:

clusterkit:
mlnx_ofed_clusterkit/clusterkit-1.15.475-1.20260211.db5c406.src.rpm

dpcp:
https://doca-repo-prod.nvidia.com/public-nsb/repo/doca/prime/generic-rpm/x86_64/dpcp/1.1.59-1/dpcp-1.1.59-1.src.rpm
ibarr:
https://doca-repo-prod.nvidia.com/public-nsb/repo/doca/prime/generic-rpm/x86_64/ibarr/2601.0.0-1/ibarr-2601.0.0-1.src.rpm
ibdump:
https://doca-repo-prod.nvidia.com/public-nsb/repo/doca/prime/generic-rpm/x86_64/ibdump/6.0.0-2/ibdump-6.0.0-2.src.rpm
ibsim:
https://doca-repo-prod.nvidia.com/public-nsb/repo/doca/prime/generic-rpm/x86_64/ibsim/0.12.1-4/ibsim-0.12.1-4.src.rpm
ibutils2:
https://doca-repo-prod.nvidia.com/public-nsb/repo/doca/prime/generic-rpm/x86_64/ibutils2/2.1.1-0.22400.MLNX202601152019.ge04c0b67f/ibutils2-2.1.1-0.22400.MLNX202601152019.ge04c0b67f.src.rpm
iser:
https://git-nbu.nvidia.com/r/a/mlnx_ofed/mlnx-ofa_kernel-4.0.git mlnx_ofed_26_01
commit 3735cf8688bf9d9bd5a425b0780a10c506abd04e

isert:
https://git-nbu.nvidia.com/r/a/mlnx_ofed/mlnx-ofa_kernel-4.0.git mlnx_ofed_26_01
commit 3735cf8688bf9d9bd5a425b0780a10c506abd04e

kernel-mft:
mlnx_ofed_mft/kernel-mft-4.35.0-159.src.rpm

libvma:
https://doca-repo-prod.nvidia.com/public-nsb/repo/doca/prime/generic-rpm/x86_64/libvma/9.8.84-1/libvma-9.8.84-1.src.rpm
libxlio:
/sw/release/sw_acceleration/xlio/libxlio-3.61.2-1.src.rpm

mlnx-ethtool:
https://doca-repo-prod.nvidia.com/public-nsb/repo/doca/prime/generic-rpm/x86_64/mlnx-ethtool/2601.0.2-1/mlnx-ethtool-2601.0.2-1.src.rpm
mlnx-iproute2:
https://doca-repo-prod.nvidia.com/public-nsb/repo/doca/prime/generic-rpm/x86_64/mlnx-iproute2/2601.0.6-1/mlnx-iproute2-2601.0.6-1.src.rpm
mlnx-nfsrdma:
https://git-nbu.nvidia.com/r/a/mlnx_ofed/mlnx-ofa_kernel-4.0.git mlnx_ofed_26_01
commit 3735cf8688bf9d9bd5a425b0780a10c506abd04e

mlnx-nvme:
https://git-nbu.nvidia.com/r/a/mlnx_ofed/mlnx-ofa_kernel-4.0.git mlnx_ofed_26_01
commit 3735cf8688bf9d9bd5a425b0780a10c506abd04e

mlnx-ofa_kernel:
https://git-nbu.nvidia.com/r/a/mlnx_ofed/mlnx-ofa_kernel-4.0.git mlnx_ofed_26_01
commit 3735cf8688bf9d9bd5a425b0780a10c506abd04e

mlnx-tools:
https://doca-repo-prod.nvidia.com/public-nsb/repo/doca/prime/generic-rpm/x86_64/mlnx-tools/2601.0.2-1/mlnx-tools-2601.0.2-1.src.rpm
mlx-steering-dump:
https://doca-repo-prod.nvidia.com/public-nsb/repo/doca/prime/generic-rpm/x86_64/mlx-steering-dump/1.0.0-1/mlx-steering-dump-1.0.0-1.src.rpm
multiperf:
https://git-nbu.nvidia.com/r/a/Performance/multiperf rdma-core-support
commit d3fad92dc6984e43cc5377ba0a3126808432ce2d

ofed-docs:
https://git-nbu.nvidia.com/r/a/mlnx_ofed/ofed-docs.git mlnx_ofed-4.0
commit 3d1b0afb7bc190ae5f362223043f76b2b45971cc

openmpi:
mlnx_ofed_ompi_1.8/openmpi-4.1.9a1-1.20260211.81d402c97a.src.rpm

opensm:
mlnx_ofed_opensm/opensm-5.26.1-202601271032.8c07ef43.tar.gz

openvswitch:
https://git-nbu.nvidia.com/r/a/sdn/ovs.git 3.3_head
commit 7b1c64c2d4109b7c98635cb7d92a57da5f143588

perftest:
https://doca-repo-prod.nvidia.com/public-nsb/repo/doca/prime/generic-rpm/x86_64/perftest/26.01.5-1/perftest-26.01.5-1.src.rpm
rdma-core:
https://doca-repo-prod.nvidia.com/public-nsb/repo/doca/prime/generic-rpm/x86_64/rdma-core/2601.0.7-1/rdma-core-2601.0.7-1.src.rpm
rshim:
/sw_mc_soc_release/packages//rshim-2.6.6-0.g0ff6d20.src.rpm

sockperf:
https://doca-repo-prod.nvidia.com/public-nsb/repo/doca/prime/generic-rpm/x86_64/sockperf/3.1-1/sockperf-3.1-1.src.rpm
srp:
https://git-nbu.nvidia.com/r/a/mlnx_ofed/mlnx-ofa_kernel-4.0.git mlnx_ofed_26_01
commit 3735cf8688bf9d9bd5a425b0780a10c506abd04e

ucx:
mlnx_ofed_ucx/ucx-1.20.0-1.20260211.d9a4f352d.src.rpm

virtiofs:
https://git-nbu.nvidia.com/r/a/mlnx_ofed/mlnx-ofa_kernel-4.0.git mlnx_ofed_26_01
commit 3735cf8688bf9d9bd5a425b0780a10c506abd04e

xpmem:
https://doca-repo-prod.nvidia.com/public-nsb/repo/doca/prime/generic-rpm/x86_64/xpmem/2601.0.9-1/dkms/xpmem-2601.0.9-1.src.rpm

Installed Packages:
-------------------

mlnx-ofa_kernel-debugsource
mlnx-tools
mlnx-ofa_kernel
mlnx-ofa_kernel-devel-debuginfo
mlnx-ofa_kernel-modules-debuginfo
xpmem-modules
xpmem
mlnx-ofa_kernel-source
mlnx-ofa_kernel-modules
mlnx-ofa_kernel-devel
kernel-mft
~~~

## NVIDIA GPU Operator

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
namespace/nvidia-gpu-operator created
operatorgroup.operators.coreos.com/nvidia-gpu-operator created
subscription.operators.coreos.com/nvidia-gpu-operator created
~~~

We can validate the operator is running with the following.

~~~bash
$ oc get pods -n nvidia-gpu-operator
NAME                            READY   STATUS    RESTARTS   AGE
gpu-operator-84db6657f7-dqrrv   1/1     Running   0          24s
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
    enabled: true
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

Now let's take the GPU cluster policy and create it on the cluster.

~~~bash
$ oc create -f gpu-clusterpolicy.yaml 
clusterpolicy.nvidia.com/gpu-cluster-policy created
~~~

After a few minutes we can validate that the pods are running.

~~~bash
$ oc get pods -n nvidia-gpu-operator
NAME                                            READY   STATUS      RESTARTS      AGE
gpu-feature-discovery-fwh9n                     1/1     Running     0             3m23s
gpu-feature-discovery-gpzws                     1/1     Running     0             3m23s
gpu-feature-discovery-mh4xq                     1/1     Running     1             3m23s
gpu-operator-84db6657f7-dqrrv                   1/1     Running     0             15m
nvidia-container-toolkit-daemonset-7zfxd        1/1     Running     0             3m23s
nvidia-container-toolkit-daemonset-mgn4v        1/1     Running     0             3m23s
nvidia-container-toolkit-daemonset-v7fxv        1/1     Running     0             3m23s
nvidia-cuda-validator-j6r2t                     0/1     Completed   0             2m12s
nvidia-cuda-validator-jbrrb                     0/1     Completed   0             2m12s
nvidia-cuda-validator-snctg                     0/1     Completed   0             2m13s
nvidia-dcgm-dkb6c                               0/1     Running     0             3m23s
nvidia-dcgm-exporter-d2fcd                      1/1     Running     3 (85s ago)   3m23s
nvidia-dcgm-exporter-s5lnr                      1/1     Running     3 (90s ago)   3m23s
nvidia-dcgm-exporter-zc8gz                      0/1     Running     2 (17s ago)   3m23s
nvidia-dcgm-mcstq                               1/1     Running     0             3m23s
nvidia-dcgm-vq6wn                               1/1     Running     0             3m23s
nvidia-device-plugin-daemonset-bmgfx            1/1     Running     0             3m23s
nvidia-device-plugin-daemonset-dxtkp            1/1     Running     0             3m23s
nvidia-device-plugin-daemonset-sgqtt            1/1     Running     0             3m23s
nvidia-driver-daemonset-10.1.20260126-0-f8q47   3/3     Running     0             4m25s
nvidia-driver-daemonset-10.1.20260126-0-mqxw8   3/3     Running     0             4m25s
nvidia-driver-daemonset-10.1.20260126-0-s7grg   3/3     Running     0             4m25s
nvidia-mig-manager-7tnmt                        1/1     Running     0             117s
nvidia-mig-manager-sj667                        1/1     Running     0             58s
nvidia-node-status-exporter-4r8bv               1/1     Running     0             4m22s
nvidia-node-status-exporter-crlcx               1/1     Running     0             4m22s
nvidia-node-status-exporter-ljbp8               1/1     Running     0             4m22s
nvidia-operator-validator-6vrbb                 1/1     Running     0             3m23s
nvidia-operator-validator-grcd5                 1/1     Running     0             3m23s
nvidia-operator-validator-z6p6w                 1/1     Running     0             3m23s
~~~

We can `rsh` into any one of the nvidia-driver-daemonset pods and execute `nvidia-smi` and show that the GPUs are reporting.

~~~bash
$ oc rsh -n nvidia-gpu-operator nvidia-driver-daemonset-10.1.20260126-0-f8q47
sh-5.2# nvidia-smi
Thu Mar 26 17:37:17 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09             Driver Version: 580.126.09     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GB200                   On  |   00000008:01:00.0 Off |                    0 |
| N/A   29C    P0            162W / 1200W |       0MiB / 189471MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   1  NVIDIA GB200                   On  |   00000009:01:00.0 Off |                    0 |
| N/A   29C    P0            149W / 1200W |       0MiB / 189471MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   2  NVIDIA GB200                   On  |   00000018:01:00.0 Off |                    0 |
| N/A   29C    P0            153W / 1200W |       0MiB / 189471MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   3  NVIDIA GB200                   On  |   00000019:01:00.0 Off |                    0 |
| N/A   29C    P0            172W / 1200W |       0MiB / 189471MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
sh-5.2# nvidia-smi topo -m
        GPU0    GPU1    GPU2    GPU3    NIC0    NIC1    NIC2    NIC3    NIC4    NIC5    NIC6    NIC7    CPU Affinity    NUMA Affinity   GPU NUMA ID
GPU0     X      NV18    NV18    NV18    SYS     SYS     SYS     SYS     SYS     SYS     SYS     SYS     0-71    0,2-17          N/A
GPU1    NV18     X      NV18    NV18    SYS     SYS     SYS     SYS     SYS     SYS     SYS     SYS     0-71    0,2-17          N/A
GPU2    NV18    NV18     X      NV18    SYS     SYS     SYS     SYS     SYS     SYS     SYS     SYS     72-143  1,18-33         N/A
GPU3    NV18    NV18    NV18     X      SYS     SYS     SYS     SYS     SYS     SYS     SYS     SYS     72-143  1,18-33         N/A
NIC0    SYS     SYS     SYS     SYS      X      SYS     SYS     SYS     SYS     SYS     SYS     SYS
NIC1    SYS     SYS     SYS     SYS     SYS      X      SYS     SYS     SYS     SYS     SYS     SYS
NIC2    SYS     SYS     SYS     SYS     SYS     SYS      X      PIX     SYS     SYS     SYS     SYS
NIC3    SYS     SYS     SYS     SYS     SYS     SYS     PIX      X      SYS     SYS     SYS     SYS
NIC4    SYS     SYS     SYS     SYS     SYS     SYS     SYS     SYS      X      SYS     SYS     SYS
NIC5    SYS     SYS     SYS     SYS     SYS     SYS     SYS     SYS     SYS      X      SYS     SYS
NIC6    SYS     SYS     SYS     SYS     SYS     SYS     SYS     SYS     SYS     SYS      X      PIX
NIC7    SYS     SYS     SYS     SYS     SYS     SYS     SYS     SYS     SYS     SYS     PIX      X 

Legend:

  X    = Self
  SYS  = Connection traversing PCIe as well as the SMP interconnect between NUMA nodes (e.g., QPI/UPI)
  NODE = Connection traversing PCIe as well as the interconnect between PCIe Host Bridges within a NUMA node
  PHB  = Connection traversing PCIe as well as a PCIe Host Bridge (typically the CPU)
  PXB  = Connection traversing multiple PCIe bridges (without traversing the PCIe Host Bridge)
  PIX  = Connection traversing at most a single PCIe bridge
  NV#  = Connection traversing a bonded set of # NVLinks

NIC Legend:

  NIC0: mlx5_0
  NIC1: mlx5_1
  NIC2: mlx5_2
  NIC3: mlx5_3
  NIC4: mlx5_4
  NIC5: mlx5_5
  NIC6: mlx5_6
  NIC7: mlx5_7
~~~

## OpenShift Virtualization

After we have the NVIDIA operators installed we need to also get the OpenShift Virtualization operator installed.
