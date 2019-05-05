# Deploy GPU-enabled Kubernetes Pod on NVIDIA Jetson Nano

NVIDIA Jetson Nano delivered GPU power in an amazingly small package. I finally got mine in the mailbox and couldn't wait to add it to my Raspberry Pi K8s cluster to take up GPU workload. It turned out however, I had to jump through a couple of hoops to get it working. In this blog post, I will walk through the steps needed and hope it will help you to get it working.

There are two things to do to enable GPU support:
- Recompile Jetson Nano's kernel to enable modules needed by Kubernetes (K8s) and in my cluster, [weaveworks/weave-kube](https://github.com/weaveworks-experiments/weave-kube), too
- Expose GPU devices to the containers running the GPU workload pods


## Recompile NVIDIA Jetson Nano's Kernel for K8s

There are many fantastic guides to recompile the kernel on Jetson devices. I would recommend reading [Hypriot blog](https://blog.hypriot.com/post/nvidia-jetson-nano-build-kernel-docker-optimized/) or [NVIDIA's forum](https://devtalk.nvidia.com/default/topic/1045726/kubernetes-node-on-jetson-tx2/), since all the steps mentioned can be done directly on Jetson Nano itself.

If you follow [Hypriot blog](https://blog.hypriot.com/post/nvidia-jetson-nano-build-kernel-docker-optimized/) and would like to use weave-kube like me, you can use a copy of the config I used here which added a few extra kernel options:
https://gist.github.com/direbearform/58349ae5ee7ddc1687b1019112746140. 

In comparison to the stock kernel, here is the list of extra options enabled:

```
CONFIG_CGROUP_HUGETLB=y
CONFIG_CFQ_GROUP_IOSCHED=y
CONFIG_INET_ESP=m
CONFIG_NF_NAT_REDIRECT=m
CONFIG_NETFILTER_XT_SET=m
CONFIG_NETFILTER_XT_TARGET_REDIRECT=m
CONFIG_NETFILTER_XT_MATCH_MULTIPORT=m
CONFIG_NETFILTER_XT_MATCH_PHYSDEV=m
CONFIG_NETFILTER_XT_MATCH_RECENT=m
CONFIG_IP_SET=m
CONFIG_IP_SET_MAX=256
CONFIG_IP_SET_BITMAP_IP=m
CONFIG_IP_SET_BITMAP_IPMAC=m
CONFIG_IP_SET_BITMAP_PORT=m
CONFIG_IP_SET_HASH_IP=m
CONFIG_IP_SET_HASH_IPMARK=m
CONFIG_IP_SET_HASH_IPPORT=m
CONFIG_IP_SET_HASH_IPPORTIP=m
CONFIG_IP_SET_HASH_IPPORTNET=m
CONFIG_IP_SET_HASH_MAC=m
CONFIG_IP_SET_HASH_NETPORTNET=m
CONFIG_IP_SET_HASH_NET=m
CONFIG_IP_SET_HASH_NETNET=m
CONFIG_IP_SET_HASH_NETPORT=m
CONFIG_IP_SET_HASH_NETIFACE=m
CONFIG_IP_SET_LIST_SET=m
CONFIG_IP_VS_PROTO_TCP=y
CONFIG_IP_VS_PROTO_UDP=y
CONFIG_IP_NF_TARGET_REDIRECT=m
CONFIG_NET_L3_MASTER_DEV=y
CONFIG_DM_BUFIO=m
CONFIG_DM_BIO_PRISON=m
CONFIG_DM_PERSISTENT_DATA=m
CONFIG_DM_THIN_PROVISIONING=m
CONFIG_IPVLAN=m
```

After rebooting, do a few modprobe to ensure that the required modules were indeed loaded (no output from modprobe means they are loaded okay):

```
sudo modprobe ip_set
sudo modprobe xt_set
sudo modprobe xt_physdev
```

Now, you can join the Jetson Nano to your existing K8s cluster.


## Create K8s pod with GPU support

As pointed [here](https://github.com/Technica-Corporation/Tegra-Docker#simple-gpu-docker-image), NVIDIA's [nvidia-docker](https://github.com/NVIDIA/nvidia-docker) is not supported on Tegra devices and they do not plan to make it so either. To workaround it, we need to give the container direct access to the GPU devices:

```
/dev/nvhost-ctrl
/dev/nvhost-ctrl-gpu
/dev/nvhost-prof-gpu
/dev/nvmap
/dev/nvhost-gpu
/dev/nvhost-as-gpu
```

We could not directly use the docker --device parameter in K8s, but we can use volumeMount as a workaround. For a simple GPU availability test, you can use device_query container image from [Tegra-Docker](https://github.com/Technica-Corporation/Tegra-Docker#simple-gpu-docker-image).

Here is an example of K8s deployment yaml to pass those devices thru, assuming you built the device_query container and [pushed to a private registry](https://forum.hilscher.com/Thread-Setup-trusted-Docker-registry-on-a-Raspberry-Pi-to-host-netPI-containers) that K8s has access to, and labeled the Jetson Nano node with <b>devicemodel=nvidiajetsonnano</b>:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gputest-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gputest
  template:
    metadata:
      name: gputest
      labels:
        app: gputest
    spec:
      hostname: gputest
      containers:
      - name: gputest
        image: <private_registry_url>/device_query:latest
        volumeMounts:
        - mountPath: /dev/nvhost-ctrl
          name: nvhost-ctrl
        - mountPath: /dev/nvhost-ctrl-gpu
          name: nvhost-ctrl-gpu
        - mountPath: /dev/nvhost-prof-gpu
          name: nvhost-prof-gpu
        - mountPath: /dev/nvmap
          name: nvmap
        - mountPath: /dev/nvhost-gpu
          name: nvhost-gpu
        - mountPath: /dev/nvhost-as-gpu
          name: nvhost-as-gpu
        - mountPath: /usr/lib/aarch64-linux-gnu/tegra
          name: lib
        securityContext:
          privileged: true
      volumes:
      - name: nvhost-ctrl
        hostPath:
          path: /dev/nvhost-ctrl
      - name: nvhost-ctrl-gpu
        hostPath:
          path: /dev/nvhost-ctrl-gpu
      - name: nvhost-prof-gpu
        hostPath:
          path: /dev/nvhost-prof-gpu
      - name: nvmap
        hostPath:
          path: /dev/nvmap
      nodeSelector:
        devicemodel: nvidiajetsonnano

```

This is the result you will see once you deploy the container and inspect its log:
```

/cudaSamples/deviceQuery Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

Detected 1 CUDA Capable device(s)

Device 0: "NVIDIA Tegra X1"
  CUDA Driver Version / Runtime Version          10.0 / 10.0
  CUDA Capability Major/Minor version number:    5.3
  Total amount of global memory:                 3964 MBytes (4156870656 bytes)
  ( 1) Multiprocessors, (128) CUDA Cores/MP:     128 CUDA Cores
  GPU Max Clock rate:                            922 MHz (0.92 GHz)
  Memory Clock rate:                             13 Mhz
  Memory Bus Width:                              64-bit
  L2 Cache Size:                                 262144 bytes
  Maximum Texture Dimension Size (x,y,z)         1D=(65536), 2D=(65536, 65536), 3D=(4096, 4096, 4096)
  Maximum Layered 1D Texture Size, (num) layers  1D=(16384), 2048 layers
  Maximum Layered 2D Texture Size, (num) layers  2D=(16384, 16384), 2048 layers
  Total amount of constant memory:               65536 bytes
  Total amount of shared memory per block:       49152 bytes
  Total number of registers available per block: 32768
  Warp size:                                     32
  Maximum number of threads per multiprocessor:  2048
  Maximum number of threads per block:           1024
  Max dimension size of a thread block (x,y,z): (1024, 1024, 64)
  Max dimension size of a grid size    (x,y,z): (2147483647, 65535, 65535)
  Maximum memory pitch:                          2147483647 bytes
  Texture alignment:                             512 bytes
  Concurrent copy and kernel execution:          Yes with 1 copy engine(s)
  Run time limit on kernels:                     Yes
  Integrated GPU sharing Host Memory:            Yes
  Support host page-locked memory mapping:       Yes
  Alignment requirement for Surfaces:            Yes
  Device has ECC support:                        Disabled
  Device supports Unified Addressing (UVA):      Yes
  Device supports Compute Preemption:            No
  Supports Cooperative Kernel Launch:            No
  Supports MultiDevice Co-op Kernel Launch:      No
  Device PCI Domain ID / Bus ID / location ID:   0 / 0 / 0
  Compute Mode:
     < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >

deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 10.0, CUDA Runtime Version = 10.0, NumDevs = 1
Result = PASS
```

## Conclusion

If you would like to create or join your new NVIDIA Jetson Nano to a K8s cluster especially with weave-kube, be prepared to recompile and deploy the kernel to enable a few options related to iptables. Once K8s cluster is created successfully, you will also need to pass the GPU devices on Jetson Nano directly into the container in the pod deployment template, given that the official NVIDIA docker plug-in does not currently support Jetson device family sporting the Tegra processor.

For detailed steps for kernel recompilation and usage of Docker and K8s, please refer to a few great blog posts linked in this write-up. I hope achieve here isto fill gaps and help to you get Jetson Nano GPU workload on K8s working end to end. Feedbacks or discussions are welcomed in case anything was not clear or did not work out well.
