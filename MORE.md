# Kube In a Box
_Disposable Kubernetes Clusters for Development_

### Background:
Ever wanted a Kubernetes cluster you could experiment with ? One where you could safely try out different ways to do things with Kubernetes, and be able to easily wipe the slate totally clean and start over when you were done? With minimal impact to your local workstation, and doesn't require you to unpick a K8S/Flannel/etc install each time to know you are back to clean? 

### Overview:

Using Vagrant, and Conjure Up this project is able to configure a full Kubernetes cluster inside a VM, and allow you to perform regular development activities against that cluster.

Sounds a bit like minikube ? Sure, except minikube installs to your workstation, and only really has the one node. This is more like running an actual kube cluster inside a VM, with lxc containers taking the place of the many physical machines you might use for a bare-metal k8s deployment. As a bonus, because it's ALL running in the VM, it doesn't mess with iptables (unlike minikube), making Kube-in-a-box able to exist with various VPN clients that make assumptions over who controls the table content.

If you just want to 'Use' Kube-in-a-box, skip down to the [Usage](#usage) Section, and read on. Otherwise, let's take a look at what it takes to put together the VM, to understand a bit more about how it all hangs together.

### How it works

Kube-in-a-box starts with Vagrant, configuring a straight Ubuntu 16.04 VM, with 2 cpus, and 8gb of ram. Our first challenge is the base image gives us 10gb of disk, when we're going to need more.

Thankfully, there's a disksize plugin for Vagrant that can fix that, but it's not installed by default. The best we can do is add some magic that explains how to install the plugin, if it's found to be missing.

With the disk size sorted, we setup the networking to be 'public' which in Vagrant terms means 'bridged' which will give the VM it's own IP on the network it comes up on. You get to pick that during `vagrant up`

We want to get the VM to the state where we can do `conjure-up kubernetes-core localhost`. That will install kubernetes for us, and do most of what we want, but it rests on a few dependencies we need to take care of first.

Ubuntu ships with the packages `lxd` and `lxd-client` installed, and while `conjure-up` is going to use lxd, it will use one installed via `snap` and having both installed at the same time gets a bit confusing, so we'll start by having Vagrant remove the Ubuntu versions from the VM.

Then we can install `lxd` and `conjure-up` via `snap install`, and we're running the "edge" version of conjure-up, because it has some output fixes, without which `vagrant up` will appear to hang for 10+minutes during bring up.

We are going to be needing docker too, which we'll get as the `docker-ce` package from the official docker repository, the docker packages in Ubuntu are older and not as up to date.

Now the system is ready for us to install Kubernetes. We ask conjure-up to do that via
>`conjure-up kubernetes-core localhost`

This will take "quite a while" (In my case, around 10-15minutes) but once complete, there will be a full kubernetes cluster running inside the VM. It's just not quite configured the way we want.. yet.

In order to be able to access the worker node, we have to do..
> `juju expose kubernetes-worker`

That allows traffic from inside the VM to reach the worker node, itself running inside an lxc container inside the VM.

We want traffic from outside the VM to reach the worker though.. which means we'll need to play with iptables a bit. Cue a bit more bash magic to figure out the IP Addresses of the Worker node, the Master Node, and our Bridged network address.

Once we know the address people will be reaching us at, (the bridged network address), we can tweak the conjure-up generated kubeconfig to tell people using it where to find us. With that done, we put the config file into the /vagrant directory, that Vagrant maps by default to the same directory the Vagrantfile was in. This means that the user running `vagrant up` will be left with a kubeconfig file they can use to talk to the cluster.

Using the external address, we can use iptables to forward any traffic from the external interface to the worker/master nodes.

We'll send port 80 & 443, and ports 30000 to 32767 to the worker node, that will allow http/https and nodeport traffic to reach the worker. And port 6443 as used by kubectl, is directed to the master node.

> `iptables -t nat -A PREROUTING -i enp0s8 -p tcp -m multiport --dport 80,443 -j DNAT --to-destination $WORKERIP`

> `iptables -t nat -A PREROUTING -i enp0s8 -p tcp --dport 30000:32767 -j DNAT --to-destination $WORKERIP`

> `iptables -t nat -A PREROUTING -i enp0s8 -p tcp --dport 6443 -j DNAT --to-destination $MASTERIP`

Lastly, we'll quickly tweak the docker daemon configuration to allow access from the external network, and launch an instance of the docker registry.

At this stage, we have a full k8s cluster running inside a bunch of lxc containers inside the VM, and docker daemon, and docker registry running just on the VM (not in an lxc container). It's helpful to think of the VM now as a set of nested worlds.. the outer shell is docker/registry/juju/conjure-up, and inside that sit the many lxc containers that represent the individual 'machines' running the constituent parts of k8s.

If you want to have a peek at whats going on, use `vagrant ssh` to access the VM. Once inside the VM, we can have a peek at whats running under the covers.

`juju status`

We used `conjure-up` to install Kubernetes, which in turn, used juju charms to perform the actual install. And juju gives us a handy way to view all the stuff we've done. Juju can deploy to real clouds, but when told to operate against localhost, it can use lxc to create the deployment. So `juju status` for a localhost install will show us all the "machines" it created, each represented by a different lxc container. Here's an example output showing etcd, easyrsa, and the kubernetes master with a single worker node.

```
ubuntu@ubuntu-xenial:~$ juju status
Model                        Controller                Cloud/Region         Version      SLA
conjure-kubernetes-core-648  conjure-up-localhost-ad9  localhost/localhost  2.3-beta2.1  unsupported

App                Version  Status  Scale  Charm              Store       Rev  OS      Notes
easyrsa            3.0.1    active      1  easyrsa            jujucharms   19  ubuntu
etcd               2.3.8    active      1  etcd               jujucharms   53  ubuntu
flannel            0.9.0    active      2  flannel            jujucharms   32  ubuntu
kubernetes-master  1.8.1    active      1  kubernetes-master  jujucharms   55  ubuntu  exposed
kubernetes-worker  1.8.1    active      1  kubernetes-worker  jujucharms   59  ubuntu  exposed

Unit                  Workload  Agent  Machine  Public address  Ports           Message
easyrsa/0*            active    idle   1        10.115.236.83                   Certificate Authority connected.
etcd/0*               active    idle   0        10.115.236.136  2379/tcp        Healthy with 1 known peer
kubernetes-master/0*  active    idle   2        10.115.236.203  6443/tcp        Kubernetes master running.
  flannel/0*          active    idle            10.115.236.203                  Flannel subnet 10.1.96.1/24
kubernetes-worker/0*  active    idle   3        10.115.236.62   80/tcp,443/tcp  Kubernetes worker running.
  flannel/1           active    idle            10.115.236.62                   Flannel subnet 10.1.43.1/24

Machine  State    DNS             Inst id        Series  AZ  Message
0        started  10.115.236.136  juju-d0c244-0  xenial      Running
1        started  10.115.236.83   juju-d0c244-1  xenial      Running
2        started  10.115.236.203  juju-d0c244-2  xenial      Running
3        started  10.115.236.62   juju-d0c244-3  xenial      Running

Relation provider                    Requirer                             Interface         Type         Message
easyrsa:client                       etcd:certificates                    tls-certificates  regular
easyrsa:client                       kubernetes-master:certificates       tls-certificates  regular
easyrsa:client                       kubernetes-worker:certificates       tls-certificates  regular
etcd:cluster                         etcd:cluster                         etcd              peer
etcd:db                              flannel:etcd                         etcd              regular
etcd:db                              kubernetes-master:etcd               etcd              regular
kubernetes-master:cni                flannel:cni                          kubernetes-cni    subordinate
kubernetes-master:kube-api-endpoint  kubernetes-worker:kube-api-endpoint  http              regular
kubernetes-master:kube-control       kubernetes-worker:kube-control       kube-control      regular
kubernetes-worker:cni                flannel:cni                          kubernetes-cni    subordinate
```

`lxc list`

Because juju is using lxc containers, we can view those directly too via `lxc list`.
Notice how you can correlate the ip addresses, and container names with the table from `juju status`. So we know from `juju status` in this example, machine 2 is running the kubernetes-master, and it has an ip of 10.115.236.203, and an id of juju-d0c244-2. When we look at our matching `lxc list` output, we can see name juju-d0c244-2 is present, and running, and has the matching ip of 10.115.236.203.
(it is also sitting on the flannel network for kubernetes, but that's an entirely different discussion)

```
ubuntu@ubuntu-xenial:~$ lxc list
+---------------+---------+--------------------------------+------+------------+-----------+
|     NAME      |  STATE  |              IPV4              | IPV6 |    TYPE    | SNAPSHOTS |
+---------------+---------+--------------------------------+------+------------+-----------+
| juju-04eeef-0 | RUNNING | 10.115.236.147 (eth0)          |      | PERSISTENT | 0         |
+---------------+---------+--------------------------------+------+------------+-----------+
| juju-d0c244-0 | RUNNING | 10.115.236.136 (eth0)          |      | PERSISTENT | 0         |
+---------------+---------+--------------------------------+------+------------+-----------+
| juju-d0c244-1 | RUNNING | 10.115.236.83 (eth0)           |      | PERSISTENT | 0         |
+---------------+---------+--------------------------------+------+------------+-----------+
| juju-d0c244-2 | RUNNING | 10.115.236.203 (eth0)          |      | PERSISTENT | 0         |
|               |         | 10.1.96.0 (flannel.1)          |      |            |           |
+---------------+---------+--------------------------------+------+------------+-----------+
| juju-d0c244-3 | RUNNING | 172.17.0.1 (docker0)           |      | PERSISTENT | 0         |
|               |         | 10.115.236.62 (eth0)           |      |            |           |
|               |         | 10.1.43.1 (cni0)               |      |            |           |
|               |         | 10.1.43.0 (flannel.1)          |      |            |           |
+---------------+---------+--------------------------------+------+------------+-----------+
```

`lxc exec`

Now we know what the lxc containers are.. we can go look inside them.. `lxc exec <containername> bash` will run bash within the selected container, letting you poke around to see whats going on under the covers. Eg. `lxc exec juju-d0c244-3 bash` in our current example, will take us into the lxc container acting as the kubernetes worker. Once inside the worker node, `docker ps` can show the containers running that kubernetes was looking after.

Being able to run a shell within the Nodes isn't the sort of thing you'll need to do if everything works as it should, but it can be really handy for debugging networking issues, (eg, confirming if a host really can't pull that docker image), or just generally for exploring what's going on within the various containers.

### Usage

Prereqs..
 - kubectl (to talk to kubernetes)
 - docker (if you want to develop/push custom images)
 - Vagrant

To use Kube-in-a-box just grab the Vagrantfile, change to the same directory, and issue `vagrant up`. After a long wait (10-15minutes for my system), it'll all be ready to use.

The VM is using bridged networking, so it will appear as a new host on your network. On that host, there's a docker registry on port 5000, the kube master on port 6443, and ports 80,443 and 30000-32767 are mapped to the kube worker node.

#### Kubernetes

When the `vagrant up` command completes, it will give info you can use to configure your environment to talk to kubernetes & docker. It will create a `kubeconfig` file in the same directory as the Vagrantfile that you can use with `kubectl` to talk to the kubernetes cluster.

Test access to your cluster using `kubectl --kubeconfig=./kubeconfig cluster-info`.

#### Docker

To work with custom images, the VM also exposes a docker daemon on port 2375 (the default), you can use this by setting the env var DOCKER_HOST to the Bridge IP. The VM will tell you the ip and port when it has finished initialising.

eg. `export DOCKER_HOST=192.168.1.100`

The VM also exposes a docker registry on port 5000. To push images so they can be used within the kube instances, tag the image with the repository name, then push the image to the repository.

eg. `docker tag your-image:yourtag 192.168.1.100:5000/your-image:yourtag`
then `docker push 192.168.1.100:5000/your-image:yourtag`

Now you can refer to the images from Kube yaml using the image name `192.168.1.100:5000/your-image:yourtag`

#### Oops?

Did something bad while experimenting? messed something up? just use `vagrant destroy` and then `vagrant up` to get a nice new clean VM to play with.

#### More than One Kube-in-a-box?

Want to run more than one Kube-in-a-box on the same host? just copy the Vagrantfile to a new directory, and edit it to alter the name `kube-in-a-box` (it appears twice in the file) to something unique

#### More than one worker node?

If you want to experiment with multiple worker nodes, you can add more using juju.

`juju add-unit kubernetes-worker`

(You can also add more etcd nodes using `juju add-unit etcd`)

#### More, More More?

Kube-in-a-box is based from the kubernetes-core juju charm. It has a good page https://jujucharms.com/kubernetes-core/ where it explains more.

kubernetes-core is the small 'development' deployment of kubernetes, there's also the canonical-kubernetes https://jujucharms.com/canonical-kubernetes/ that has a lot more components.. (but also requires a lot more resource, so be prepared to update the VM resources)
