# Vagrant for Kubernetes app development in a VM.

This sets up a _public_ network (you'll get prompted for which of your NICs should be used). This emulates a public-routable VM. Take precautions (tho testing the precautions is somewhat the point here)

Once up and running, 

Port mappings:

| From                     |    | To                         |
| ------------------------ | -- | -------------------------- |
| Bridge IP : 80,443       | -> |  Worker Node : 80,443      |
| Bridge IP : 30000-32767  | -> |  Worker Node : 30000-32767 |
| Bridge IP : 6443         | -> |  Master Node : 6443        |
| Bridge IP : 2375,5000    | -> |  VM Docker Daemon / Registry | 
  
After `vagrant up` has completed, there will be a full kube running inside the VM with 5 'machines' running (each machine is an lxc container), providing easyrsa, etcd, flannel, kubernetes-master and a single kubernetes-worker.

## Kubernetes

A config file (called `kubeconfig`) to let you talk to the kube will be generated into the same directory as the Vagrantfile. 
Test access to your cluster using `kubectl --kubeconfig=./kubeconfig cluster-info` 

## Docker

To work with custom images, the VM also exposes a docker daemon on port 2375 (the default), you can use this by setting the env var DOCKER_HOST to the Bridge IP. The VM will tell you the ip and port when it has finished initialising. 

eg. `export DOCKER_HOST=192.168.1.100`

To push images so they can be used within the kube instances, tag the image with the repository name, then push the image to the repository. 

eg. `docker tag your-image:yourtag 192.168.1.100:5000/your-image:yourtag`
then `docker push 192.168.1.100:5000/your-image:yourtag`

Now you can refer to the images from Kube yaml using the image name `192.168.1.100:5000/your-image:yourtag`

## More

If you want to see further details.. 

First, `vagrant ssh` to access the VM hosting the machines. 
then `juju status` will show the status of the processes on those machines.
`iptables -L -vt nat` will show the forwarding rules added to the host.
`lxc list` will show the running containers.


