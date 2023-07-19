---
title: Configuring GPUs and running GPU workloads on RKE2 (2023 edition)
author: Brian Durden
email: brian.durden@ranchergovernment.com
---
# Configuring GPUs and running GPU workloads on RKE2 (2023 edition)

## Introduction
This writeup and associated repo will be built for configuring RKE2 to handle GPUs and it will take into account developments that have occured in the last year or so that make doing so significantly easier to do both manually as well as tying into automation pipelines. I'll try to cover each subject contextually so you can make the best choice for your solution.

---

> **Table of Contents**
> * [Hardware Considerations](#hardware-considerations)
> * [Software Considerations](#software-considerations)
> * [Nvidia Operator Howto](#nvidia-operator)
> * [Metrics / Observability in Prometheus (via Rancher Monitoring too)](#metrics--observability)
> * [Running Workloads](#running-workloads)

---

## Hardware Considerations

 -- TODO: insert Nvidia logo --
Nvidia is going to be the subject of this doc, while it is possible to get AMD's GPUs up and working in RKE2 they are definitely less further along to the process than Nvidia and it will still require a good deal of manual effort.

Nvidia has essentially two different classifications of GPUs, I like to distinguish them as consumer and enterprise grade cards. On the bright-side, the consumer-grade cards are an AMAZING value for doing infrastructure testing and even development/POCs on. Many of them are quite fast and/or have significant amounts of VRAM available to do a lot of work. An RTX3060 card can be had with 12gb VRAM for relatively cheap (~$350 as of this writing) and makes for a great card running Stable Diffusion

Where Enterprise-grade cards shine the most is their featureset. Consumer cards are great at running single workloads but need significant assistance in order to share workloads. When running in Kubernetes like RKE2, being able to share multiple workloads across multiple nodes is one of the biggest reasons enterprise cards have become so popular. So not having some of these capabilities can prevent consumer cards from being a viable option in an enterprise and/or production deployment. This is aside from the fact that the enterprise cards come with significantly more capacity for gpu-compute and vram in addtion to better heat tolerance and longevity.

-- TODO: insert chart with nvidia enterprise cards --

So when considering hardware, consider what you're trying to do. You cannot use MIG on consumer cards and even on many still-viable enterprise-grade cards, so sharing many workloads on a single GPU is just not possible in a memory/compute-secure manner (timeslicing is a different subject and isn't secure but that may not matter for you). If you don't need MIG and/or have many GPUs to share on multiple hosts, then you're good.

## Software Considerations

Diving into software for a minute, let's address when/where these workloads are going to run. The subject of this doc is going to be working within RKE2 which uses containerd by default as do most Kubernetes distributions. If you are using docker, then you are on your own as I haven't tested using docker on RKE2 and frankly the industry has moved away from docker running containers in Kubernetes. I tend to go where my customers are in terms of infrastructure dependencies and very very few are using docker in any capacity beyond building container images or the odd docker-compose on a VM somewhere.

RKE2 is Kubernetes and as such it is designed to abstract away the infrastructure running underneath it. This means RKE2 will function with nvidia GPUs as I describe whether you are running bare metal or hypervisor. If the PCI devices can get direct mapped to the RKE2 worker node(s) then you should meet success pretty easily when running in a hypervisor-based infrastructure. Bare metal is certainly easier to test but obviously significantly harder to automate as provisioning a bare metal machine is not a 'solved problem' quite yet even though there are solutions out there.

This doc assumes you have installed an RKE2 cluster on a typical 1/3 configuration for simplicity, so one control-plane node and 3 workers. You don't HAVE to have 3 workers, but that's just what I'm assuming here and you'll know why soon.

With a base configuration (without MIG), you are limited to physical GPUs mapped to each node. So if you have 3 workers and 1 GPU, only one of those workers can have a GPU available. This is a realistic scenario in most enterprise cases too because Kuberentes allows us to handle heterogeneous workloads based on its labeling and taint/toleration capabilities. After we're done with the operator install, Kubernetes will be able to map GPU workloads to the correct RKE2 nodes. However, the only way to run more than one workload (pod) using a GPU without significant 3rd party modification is to add another GPU.

-- insert graphic showing gpu to pod mapping --

With MIG, a single GPU can be sliced into multiple virtual devices (with their own secure memory space and compute) and be available to different pods, effectively sharing the GPU. Combined with multiple GPUs per node as well as multiple nodes with GPUs, the scalability of MIG is quite significant.

-- insert MIG graphic --

### Driver considerations

Nvidia's Linux drivers are quite easy to install regardless of your distribution (within reason). This is a consideration based on the nature of the Operator. The operator itself will install the driver if directed, but I have personally seen issues with the driver installation on RHEL and Rocky OS installations. Given how IBM/RHEL has gone closed-source, I wouldn't expect a community fix for this anytime soon. And due to the kernel-modification nature of the driver, some security teams require major changes to VM instances like that to go through vetting. This isn't just containers here, this is the actual driver as well as a specialized nvidia-containerd runtime.

Given the above and that I typically want a generalized solution that works for all, I usually install the driver as part of the VM provisioning process as opposed to letting the operator do it. Within Rancher MCM this is an easy `cloud-init` line, but the Nvidia drivers for most OS's are already handled by the OS-level package manager (`yum`,`rpm`,`dpkg`,`apt`,`zypper`). The driver itself is tightly coupled to the container image versions of each component in the operator. So when doing upgrades, this is something that will have to be watched closely.

For Ubuntu and SLES, this can be done using the `nvidia-driver-XYZ` package and may require a reboot. Using the `nvidia-smi` tool on the command-line can allow one to see the available GPUs.

-- insert example of nvidia-smi --

## Operator installation
The Nvidia operator is a K8S installable that will follow on with an install of a set of Nvidia tools designed to expose the GPU for containerized workloads in a Kubernetes-native way. It has a significant amount of features and they are exposed within the helmchart. The way we reference the operator here will cover the helmchart method. It has received significant upgrades in recent years which has made a lot of previous howtos around Nvidia GPUs and RKE2 or K8S obsolete. The biggest change is the capability of automatically modifying the containerd configuration on the node as well as installing the nvidia containerd runtime, both of which were manual efforts. This drastically reduces the amount of pre-setup for a GPU-based node, essentially making it no different than a regular node other than the PCI device mapping (or in AWS, you just ask for an EC2 instance with a gpu on it)

Below I'm going to cover changes that need to happen to the helm chart values file. Nvidia's comments are using the `#` notation, but mine will be the `#!` notation so we can see my comments inline. I have two helmchart values files in this repo, one is the default one from upstream Nvidia, the second is my modified one.

### Containerd modifications
The Nvidia operator expects the `containerd.toml` file and the `sock` file to be located in their default places. RKE2 doesn't use default vanilla K8S file locations, so we need to set those values. 

```yaml
toolkit:
  enabled: true
  repository: nvcr.io/nvidia/k8s
  image: container-toolkit
  version: v1.13.0-ubuntu20.04
  imagePullPolicy: IfNotPresent
  imagePullSecrets: []
  env: []  #! Here we set the containerd values using environment vars 
  #! env:
  #! - name: CONTAINERD_CONFIG
  #!   value: /var/lib/rancher/k3s/agent/etc/containerd/config.toml
  #! - name: CONTAINERD_SOCKET
  #!   value: /run/k3s/containerd/containerd.sock
  #! - name: CONTAINERD_RUNTIME_CLASS
  #!   value: nvidia
  #! - name: CONTAINERD_SET_AS_DEFAULT
  #!   value: 'true'
  #! resources: {}
  #! installDir: "/usr/local/nvidia"
```

### Driver Disable
As stated above, I'm installing the driver as a linux package on my base VMs. Because of that, I'm disabling the driver installer component in the operator:

```yaml
driver:
  enabled: true  #! Here we set the driver to disabled
  #! enabled: false   
```

### PSP
If you're using the cis-1.6 profile in RKE2, one of the changes it makes is in PSPs. 1.25 need not apply and uses a different mechanism. If you're using pre 1.25 RKE2 and have a secure or stig-compliant cluster, you'll need psp enabled

```yaml
psp:
  enabled: false  #! if you are using cis-1.6 profile (which you should be in a secured environment), you'll need the PSPs to get generated here. Unsure of 1.25 implications here
  #! enabled: true 
```

### MIG
By default the MIG manager is enabled and leaving it on will not affect non-MIG compliant devices as MIG-enabled devices at the driver level show up as physical interfaces. One thing to be aware of is in a multi-GPU arrangement on a single node. The mig strategy needs to be considered. Read [here](https://docs.nvidia.com/datacenter/cloud-native/kubernetes/latest/index.html#using-mig-strategies-in-kubernetes) for more information. I am leaving MIG ignored for now as I have no way of verifying various tweaks easily.


### Helm Installation
Using helm, we can install the operator using the values file we have built. Ensure your kube context is pointed at a valid RKE2 instance with at least one GPU node.

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia![Alt text](https://files.slack.com/files-pri/T051C3C2NBV-F05HC8U5MRV/screenshot_2023-07-19_at_10.53.56_am.png)
helm install nvidia/gpu-operator -f operator_values_modified.yaml
```

* what to look for determining success/fail
* secondary tests
* mentions around MIG

## Metrics / Observability

* mention of DCGM exporter
* tying to Rancher Monitoring
* example of helm chart config
* example of nvidia grafana dashboard (https://grafana.com/grafana/dashboards/6387-gpus/)

## Running Workloads
This is probably the most discussed topic that I get questioned on. Once you've got the GPUs up and running, how do you PROVE they can do actual GPU-based work? Nvidia doesn't provide much help there and to be honest their test code using things like `vectoradd` is vastly sub-par on many levels. 

The biggest gripe I have with `vectoradd` and other examples is that while it is a container that Nvidia publishes, it is tightly coupled to the version of the driver you are using on the node. This breaks the Kubernetes/app-platform abstraction making a pod explicitly dependent on a very specific configuration at the node level. Many times `vectoradd` does not support the newest driver or whatever driver version a customer is using so they are forced to downgrade their drivers in order to run a test. This is silly.

The second biggest gripe is that `vectoradd` and other examples aren't really taxing the GPU at all. It's like having a racecar with track-ready suspesion, brakes, and huge powerband and 'testing' it by rolling around a parking lot. 

-- insert racecar parked at grocery store image --

In the now-exploding AI market there are quite a few toolsets out there leveraging PyTorch and other apps in order to deliver outputs from AI modeling. The two most notorious at the time of publishing is ChatGPT and StableDiffusion/Midjourney. While we're still a ways off from ChatGPT running on RKE2 as the hardware requirements for it are reportedly HUGE, StableDiffusion is perfectly capable of running on a local machine with a single consumer-grade GPU. I found a semi-containerized version of an SD UI and was able to port it and run it in RKE2 on top of Harvester using a simple Ryzen9-based miniPC along with an RTX3060 GPU. It didn't always function perfectly as the front-end UI was not designed to work over a high-latency web-app interface. Now, 8 months later, the market has accelerated and there are other more mature container apps available now.

I recommend beginning your journey working to get Automatic1111 up and running as a K8S container. As this doc develops, I will go step by step on how I made that happen. But the beginnings are [here](https://github.com/AbdBarho/stable-diffusion-webui-docker/tree/master). This particular setup is designed around a docker-compose scheme, and while not exactly K8S-native, under the hood it builds the containers we need and can use. Everything here describes both how to make the containers, their source images, as well as ports/access that has to happen. So I will be working from this starting point.