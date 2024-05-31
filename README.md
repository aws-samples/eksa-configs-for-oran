# EKS Anywhere - COTS Hardware BIOS settings, Plugins Installation and Configurations needed for O-RAN CU/DU onboarding

This page describes all the pre-requisites - BIOS, Kernel settings on the EKS Anywhere (EKS-A) HPE DL110 workers, other plugin packages and configurations on EKS Anywhere needed for O-RAN CU/DU onboarding.

[[_TOC_]]

## 1. HPE Hardware BIOS settings for RAN Real-time performance

HPE DL110 Bare metal servers must be configured for maximum performance, low latency and high throughput. Thus the HPE Server BIOS is changed to enabled the following parameters -

| Parameter         | Configuration |
|-------------------|---------------|
| Clock Frequency   | Fixed         |
| C-States          | Disabled      |
| Turbo Power boost | Disabled      |
| Hyper Threading   | Enabled       |
| Virtualization    | Disabled      |
| LLC Prefetch      | Enabled       |
| SR-IOV            | Enabled       |

Detailed screenshot of BIOS settings are in the following link - [Bios Settings](1-bios-settings.md)

## 2. Ubuntu OS Performance Tuning, CPU isolation & Pinning  (for Real-time performance)

RAN application require dedicated CPUs assigned for Real-time performance, thus it is essential to give RAN application the most execution time. This can be achieved by isolating CPU's - thereby excluding these CPU's for any other user workload scheduling/execution. CPU isolation generally means allocation of separate CPU’s for the following -

* OS System processes (Init & system daemons)
* User processes
* Kubernetes workload processes

### 2.1. GRUB config for CPU isolation

This CPU isolation could be achieved by configuring specific parameters in the Ubuntu OS grub (`/etc/default/grub`) as shown in the below example. In this example, On an HPE DL110 server with 64 vCPU - we could have dedicated 6 vCPU's - 0,1,2,32,33,34 for System and User space and the remaining vCPU's (3-31,35-63) for the Kubernetes RAN Application workloads.

```sh
kthread_cpus=0,1,2,32,33,34
irqaffinity=0,1,2,32,33,34
isolcpus=managed_irq,3-31,35-63
nohz=on
nohz_full=3-31,35-63
rcu_nocbs=3-31,35-63
rcu_nocb_poll
intel_iommu=on
iommu=pt
vfio_pci.enable_sriov=1
```

### 2.2 GRUB config for Hugepages and Real-time processing kernel parameters

RAN workloads require the use of hugepages for improved performance, the following example configuration shows the grub config (`/etc/default/grub`) for hugepages.

```sh
hugepagesz=1G
hugepages=30
default_hugepagesz=1G
```

For Real-time processing it is important that the RAN workloads have full access to the CPU bandwidth for scheduling of Real-time tasks.
To achieve the same the parameter `/proc/sys/kernel/sched_rt_runtime_us` must be set to `-1` so that Real-time tasks may use up to 100% of CPU times.

To configure this parameter, follow the below instructions.

```sh
vi /etc/sysctl.conf
```

Add the following line at the end.

```
kernel.sched_rt_runtime_us = -1
```

Apply parameter.

```sh
sysctl -p
```

Check the parameter.

```sh
cat /proc/sys/kernel/sched_rt_runtime_us
-1
```

Additional details on Real-time kernel on Ubuntu are described in the following Canonical Ubuntu [blog](https://ubuntu.com/blog/Real-time-kernel-tuning).

### 2.3. Cgroup v2 CPU isolation configurations

Additionally, CPU isolation configurations are needed on the Ubuntu OS by use of Cgroup v2 cpusets. The following commands are an example of how this can be configured on the Ubuntu OS -

Set cpuset for `system.slice` and `user.slice`.

```sh
systemctl set-property system.slice AllowedCPUs=0,1,2,32,33,34
systemctl set-property user.slice AllowedCPUs=0,1,2,32,33,34
```

For `kubepods.slice`, the configuration file is not automatically created, so create the file manually.

```sh
systemctl set-property kubepods.slice AllowedCPUs=3-31,35-63
mkdir -p /etc/systemd/system.control/kubepods.slice.d/
cat > /etc/systemd/system.control/kubepods.slice.d/50-AllowedCPUs.conf << EOF
[Slice]
AllowedCPUs=3-31,35-63
systemctl set-property kubepods.slice AllowedCPUs=3-31,35-63
```

For `init.scope`, the configuration file based settings does not work, so it must be added the command to rc.local.

```sh
systemctl set-property init.scope AllowedCPUs=0,1,3,32,33,35
echo "systemctl set-property init.scope AllowedCPUs=0,1,3,32,33,35" >> /etc/rc.local
```

### 2.4. Kubernetes CPU Manager CPU isolation configurations

Additionally with Kubernetes CPU manager - CPU isolation configuration can be done to ensure Kubelet allocates exclusive CPUs to certain pod containers. Kubernetes CPU manager (`/var/lib/kubelet/config.yaml`) must be configured with “static” policy on each of the worker nodes of the cluster.

```yaml
cpuManagerPolicy: static
cpuManagerPolicyOptions:
  full-pcpus-only: "false"
cpuManagerReconcilePeriod: 5s
topologyManagerPolicy: best-effort
reservedSystemCPUs: 0,1,2,32,33,34
```

### 2.5. irqbalanced CPU isolation

The following configuration is needed only for Real-time performance therefore applicable only for DU EKS-A worker nodes. Edit irqbalanced configuration file as follows and configure the list of CPU's that are reserved for the system space.

```sh
vi /etc/default/irqbalance
```

```sh
#IRQBALANCE_BANNED_CPULIST=
IRQBALANCE_BANNED_CPULIST=3-31,35-63
```

Restart the daemon.

```sh
systemctl daemon-reload
systemctl restart irqbalance
```

## 3. SR-IOV/DPDK Plugin Installation on EKS Anywhere

Intel Data Plane Development Kit (DPDK) comes with a set of libraries and drivers need for fast packet processing. These libraries can be built and installed on EKS-A worker nodes using the following example commands -
The steps involve downloading the DPDK library archive, install Go library for the build, creating SR-IOV Virtual Functions (VFs) and binding them to the corresponding network interface.

### 3.1. DPDK installation on EKS-A worker nodes

Please follow below steps to download DPDK archive, build and install them on the EKS-A worker nodes.

```sh
sudo su
apt-get -y update
apt-get -y --allow-change-held-packages install git libnuma-dev libhugetlbfs-dev build-essential cmake meson pkgconf python3-pyelftools

# with root-permission
cd /opt/
wget http://static.dpdk.org/rel/dpdk-21.11.tar.xz
tar xf /opt/dpdk-21.11.tar.xz

cd /opt/dpdk-21.11
meson build
ninja -C build
ninja -C build install

# check DPDK
cd usertools/
/opt/dpdk-21.11/usertools/dpdk-devbind.py -s
```

### 3.2. SR-IOV CNI Plugin installation

DPDK uses the SR-IOV network for hardware-based I/O sharing. Following steps provide commands for the installation of SR-IOV Device plugin on the EKS-A worker node. Please note that the SR-IOV binary build require “go” packages as a pre-requisite.

```sh
sudo -i
cd
wget https://go.dev/dl/go1.16.linux-amd64.tar.gz
tar -xvf go1.16.linux-amd64.tar.gz -C /usr/local/
export PATH=$PATH:/usr/local/go/bin
# check by go env
go env

git clone https://github.com/k8snetworkplumbingwg/sriov-cni.git
cd sriov-cni
git checkout v2.2
mkdir bin
# download and install golint
go get -u -v golang.org/x/lint/golint
cp ~/go/bin/golint bin/
go env -w GO111MODULE=off
make
cd build
cp sriov /opt/cni/bin
```

### 3.3. SR-IOV Virtual Function creation and binding

The SR-IOV Network Device Plugin requires SR-IOV virtual functions (VFs) to be created on the EKS-A worker node.
e.g. To create 4 virtual functions on a physical interface (PF_NAME), run the following command.

```sh
echo 4 > /sys/class/net/${PF_NAME}/device/sriov_numvfs
```

VFs creation can be validated using following command.

```sh
lspci | grep "Virtual Function"
6c:19.0 Ethernet controller: Intel Corporation Ethernet Adaptive Virtual Function (rev 02)
6c:19.1 Ethernet controller: Intel Corporation Ethernet Adaptive Virtual Function (rev 02)
6c:19.2 Ethernet controller: Intel Corporation Ethernet Adaptive Virtual Function (rev 02)
6c:19.3 Ethernet controller: Intel Corporation Ethernet Adaptive Virtual Function (rev 02)
```

Binding VFs to DPDK vfio-pci drivers.

```sh
/opt/dpdk-21.11/usertools/dpdk-devbind.py -b vfio-pci 6c:19.0 6c:19.1 6c:19.2 6c:19.3
```

Check if the vfio devices are bound against the DPDK driver using the following command.

```sh
/opt/dpdk-21.11/usertools/dpdk-devbind.py -s
```

### 3.4 SR-IOV Network Device Plugin installation

Make a clone copy of the SR-IOV Network Device Plugin repo in the [GitHub](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin).

```sh
git clone https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin
```

Create SR-IOV ConfigMap on the EKS Anywhere cluster along with VF information from the previous step. Please remember the pfName to the appropriate SR-IOV physical interface name.

```sh
cat <<EOF > sriovdp-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sriovdp-config
  namespace: kube-system
data:
  config.json: |
    {
        "resourceList": [{
            "resourceName": "intelnics01",
            "resourcePrefix": "eks-a.io",
            "selectors": {
                "vendors": ["8086"],
                "devices": ["154c", "10ed", "1889"],
                "drivers": ["vfio-pci"],
                "pfNames": ["PF_NAME"]
            }
          }
        ]
    }
EOF
```

Apply the SR-IOV ConfigMap and create the SR-IOV DaemonSet using the following commands.

```sh
kubectl apply -f sriovdp-configmap.yaml
kubectl apply -f sriov-network-device-plugin/deployments/sriovdp-daemonset.yaml
```

Validate that the SR-IOV VF’s are applied against the EKS-A worker node using the following command.

```sh
kubectl get node <eksa-node-name> -o json | jq '.status.allocatable'
{
  "cpu": "64",
  "eks-a.io/intelnics01": "4",
  "ephemeral-storage": "424730208372",
  "hugepages-1Gi": "30Gi",
  "hugepages-2Mi": "0",
  "memory": "100099604Ki",
  "pods": "110"
}
```

Detailed instructions for setup of SR-IOV Network Device Plugin for Kubernetes is described in the following [GithHub](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin).

## 4. Multus CNI Plugin installation on EKS Anywhere Cluster

Installation of Multus plugin is required on the EKS Anywhere cluster when the RAN workloads needs additional Multus interfaces for network separation requirements.
The following commands could be used for the Multus DaemonSet installation on the EKS Anywhere cluster.

```sh
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml
kubectl get daemonsets.apps -n kube-system
```

Detailed instruction on Multus CNI plugin installation and Multus NetworkAttachmentDefinition CRD configuration for IPVLAN type is described in the following [link](https://anywhere.eks.amazonaws.com/docs/clustermgmt/networking/cluster-multus/).

## 5. Disable Chrony on EKS-A Worker assigned for DU workload

### 5.1. Disable Chrony service

Precision Timing Protocol (PTP) is required for the DU workload to get more accurate timing information for O-RAN components (e.g. RU). DU application installs PTP plugin on EKS-A and to avoid the timing source conflicts between PTP and NTP (default on the platform), we need to disable Chrony service on the platform with the following commands.

```sh
systemctl stop chrony
systemctl disable chrony
systemctl status chrony
```

### 5.2. Create one-shot Chrony service

Create a service to synchronize the time only once at startup, since stopping Chrony can cause the time to shift.

```sh
cat > /etc/systemd/system/chrony-once.service << EOF
[Unit]
Description=Update system time using chronyc before starting kubelet service
Before=kubelet.service

[Service]
Type=oneshot
ExecStart=/usr/sbin/chronyd -q "server ntp.ubuntu.com iburst"

[Install]
WantedBy=multi-user.target
EOF
```

```sh
systemctl daemon-reload
systemctl enable chrony-once.service
```
