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
to each other and the Internet once they are booted.Considering that we are going to build a computing cluster with about 100
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

## CoreOS on VirtualBox

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

I tried to follow https://github.com/emergingstack/es-dev-stack/ to
build a Docker image of CUDA kernel module.  It builds.  But when I
run the built docker image, it complains that the system doesn't have
CUDA GPU. (Yes, it is a VM that doesn't have any GPU).

## CoreOS on AWS EC2

It is true that we can create CoreOS instances as explained in
[this post](http://tleyden.github.io/blog/2014/11/04/coreos-with-nvidia-cuda-gpu-drivers/),
but I cannot find a way to ssh to the instance.  So I resorted to
CloudFoundation to create a CoreOS cluster with at least 3 instances.
CoreOS's tutorial lacks details, reasons, explanations.  Luckily, I
found [this tutorial](https://deis.com/blog/2016/coreos-on-aws/).

### Too Big Docker Image to Build

It is notable that the Dockerfile will run out of disk space on either
`g2.x2large` nodes or `g2.8xlarge` nodes.  So I tried to git clone
only the most recent commit of the wanted branch of Linux kernel code:

1. `git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git linux` gives 1.8GB.
1. Then I removed the `.git` subdirectory. It leaves 715MB.
1. `git clone -b v4.4.8 --depth 1 git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git linux` gives 850MB
1. Then I removed the `.git` subdirectory. It leaves 696MB.

Anyway, I cannot build the Docker image on even `g2.8xlarge`.

So I go back to use VirtualBox VM and installed CoreOS 899.15.0.
After ssh into the VM, `uname -r` shows Linux kernel version 4.3.6 --
exactly as that on AWS!

So I build the Docker image on the VM, tag and push it:

```
docker tag xxxxxx cxwangyi/cuda:v1
docker login --username=cxwangyi --email=yi.wang.2005@gmail.com
docker push cxwangyi/cuda
```

Then on an EC2 CUDA instance, I run:

```
docker run -it --priviledged cxwangyi/cuda:v1
```

### Too Big Docker Image to Pull

However, this complains that the Docker image to be pulled is too big
and runs out of disk space.  The question becomes: how to make Docker
images smaller.  Google told me --
[docker-squash](https://github.com/jwilder/docker-squash).

I downloaded pre-built docker-squash from its Github README.md page,
and scp to my VirtualBox VM, untar there.  Then I followed the
[usage description](http://jasonwilder.com/blog/2014/08/19/squashing-docker-images/)
to remove Linux kernel source code and NVidia packages and
docker-squash.

On my VM and AWS EC2 instances, CoreOS mounts `/tmp` to a small
in-memory filesystem which is too small.  The solution is simply `sudo
umount /tmp` so that `/tmp` is on the big disk partition which holds
`/`.


### Solution

The solution is simply build the Docker image manually, so all changes
get into a single commit.  During this manual procedure, we delete
files that are no longer useful.  This starts from the base image of
ubuntu:14.04:

```
docker run -it --privileged ubuntu:14.04 /bin/bash
```

followed by all steps enlisted
[here](https://github.com/wangkuiyi/es-dev-stack/blob/clone-most-recent-commit-of-linux-kernel/corenvidiadrivers/Dockerfile)
with RUN and the final CMD.  The final step generates
`/opt/nvidia/nvidia_installers/NVIDIA-Linux-x86_64-352.39/kernel/uvm/nvidia-uvm.ko`,
the kernel module we need.  Then all other stuff, Linux kernel source
code and CUDA things can be deleted.  Then we do

```
exit # exit from the running of /bin/bash in ubuntu:14:04 container
docker commit <id>
```

so to make all our work into a single Docker commit.  The `docker
commit` command will print a new Docker commit id, say <new_cid>,
which can be tagged:

```
docker tag cxwangyi/cuda <new_cid>
```

An alternative solution is to combine all RUN directives in Dockerfile
into a single one, so that `docker build` creates a single commit with
all changes.

### Pitfalls

I tried using docker-squash to reduce the image size.  But it is not
as effective as grouping all changes into a single Docker commit.

I once forgot to use the `--privilege` option and causes some
confusions as I documented
[here](https://github.com/emergingstack/es-dev-stack/issues/15).
