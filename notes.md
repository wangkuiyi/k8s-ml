# CoreOS + CUDA + Kubernetes + Tensorflow

I used to think that Ceph and Kubernetes should be installed on bare
metal servers running Ubuntu Linux with the CUDA driver.  But
according to a very recent
[post](http://www.emergingstack.com/2016/01/10/Nvidia-GPU-plus-CoreOS-plus-Docker-plus-TensorFlow.html),
it seems that CUDA driver can be built into Linux kernel modules and
insert into a Docker image together with Tensorflow, so that when we
run the image the CUDA kernel module is loaded by the kernel.  This
leads to an interesting new solution -- we can run everything, from
CUDA driver to Kubernetes and Ceph in containers.  Then we don't need
Ubuntu at all; instead we can use CoreOS.


A key point with above solution that runs CUDA kernel modules within a
container is that when we build the kernel module, we must use the
same version of Linux kernel source code as the CoreOS.  This is
achieved by checking the host (CoreOS) Linux kernel version in the
[Dockerfile](https://github.com/emergingstack/es-dev-stack/blob/master/corenvidiadrivers/Dockerfile):

```
RUN git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git linux
RUN git checkout -b stable v`uname -r | sed -e "s/-.*//" | sed -e "s/\.[0]*$//"`
```

## Computing Cluster

With the new idea of running everything on top of CoreOS, the first
step is installing CoreOS.  Before that, we need to build a computer
cluster.  I consulted Bryan Huang, my former colleague at Google IT
team.  Bryan and his friend 吕宏利, a Google engineer, helped me
understand the general network structure of a in-house cluster:

- Every 16/24 computers are connected by cables to a switch.
- Every some switches are connected by cables to a higher level switch.
- A router with two ends connects the top switch with the Internet.

As long as we configure computers to use HDCP and configured the
router and all switches, those computers should be able to communicate
to each other and the Internet once they are booted.

Considering that we are going to build a computing cluster with about 100
nodes, some recommended hardwares:

- a [router with firewall](http://www.fortinet.com.cn/products/fortigate/60D.html),
- 16 ports 100M
  [switch](http://www.amazon.com/NETGEAR-ProSAFE-FS116NA-16-Port-Ethernet/dp/B000063UZW/ref=sr_1_11?ie=UTF8&qid=1462245958&sr=8-11&keywords=switch)
  or 1000M switch
  (http://www.amazon.com/NETGEAR-16-Port-Gigabit-Ethernet-Desktop/dp/B01AX8XHRQ/ref=sr_1_1?ie=UTF8&qid=1462246013&sr=8-1-spons&keywords=switch+netgear&psc=1)
  depending on communication workload.

To better understand the tree-toplogy of the network, it is important
to understand two concepts
[广播域和冲突域](http://202.201.18.40:8080/mas5/bbs/showBBS.jsp?id=5015&forum=259&noButton=yes).

## CoreOS

吕宏利 reminded that even if it is true that we can boot servers from
PXE server, the PXE server is a single-point -- if it fails, the whole
cluster wouldn't be able to boot.  So it is more reasonable to boot
CoreOS using PXE, and install CoreOS into disks.

I followed
[this tutorial](https://gist.github.com/noonat/9fc170ea0c6ddea69c58)
and successfully installed CoreOS into disks of a VirtualBox VM.  The
general idea is that I boot the VM using a CoreOS ISO image, and run a
script `coreos-install` in that image to install CoreOS into the
specified disk, say `/dev/sda`.  The script requires a config file,
which should contain at least a user's SSH public key, so that the
user can ssh to the CoreOS VM later.  I put this config file into my
Github repo, and CoreOS has basic network tools like curl, which can
be used to download the config file for use by `coreos-install`.
