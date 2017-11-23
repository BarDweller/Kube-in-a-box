[
  { :name => "vagrant-disksize", :version => ">= 0.1.2" }
].each do |plugin|
  if not Vagrant.has_plugin?(plugin[:name], plugin[:version])
    raise "#{plugin[:name]} #{plugin[:version]} is required. Please run `vagrant plugin install #{plugin[:name]}`"
  end
end

Vagrant.configure("2") do |config|
#
# Notes:
# Using ubunutu, with bridged networking, should mean the bridge is always called enp0s8
# if not, a lot of the references here will be meaningless.
#
  config.vm.box = "ubuntu/xenial64"
  config.disksize.size = '40GB'

  config.vm.define "k8s-local-rbac-snapshot" do |vm|
  end

  config.vm.provider "virtualbox" do |vb|
   vb.cpus = 2
   vb.memory = "8192"
   vb.name = "k8s-local-rbac-snapshot"
  end

  config.vm.network "public_network"

  config.vm.provision :shell, :inline => <<-EOT
    apt-get update
    apt-get remove -qyf lxd lxd-client
    DEBIAN_FRONTEND=noninteractive apt-get -y -o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold" upgrade

    snap install lxd
    snap install conjure-up --classic --edge
    snap install kubectl --classic

    usermod -a -G lxd ubuntu

    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
    add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    apt-get update
    apt-cache policy docker-ce
    apt-get install -y docker-ce socat

    usermod -aG docker ubuntu

  EOT

  config.vm.provision :shell, privileged: false, :inline => <<-EOT

    echo "starting conjure-up for kube, this takes quite a while"
    date
    conjure-up kubernetes-core localhost
    echo "done"
    date
   
    if [ ! -d "~/.kube" ]; then 
      echo "recovering kube config. (failed start?)"
      mkdir -p ~/.kube
      juju scp kubernetes-master/0:config ~/.kube/config
      cp ~/.kube/config ~/.kube/config.conjureup
    fi

    echo "reconfiguring kube for rbac"
    juju config kubernetes-master allow-privileged=true 
    juju config kubernetes-worker allow-privileged=true
    juju ssh kubernetes-master/0 -- 'sudo snap set kube-apiserver authorization-mode=RBAC'
    echo "enabling k8s external admission hooks (required by istio)"
    juju config kubernetes-master api-extra-args="admission-control=Initializers runtime-config=admissionregistration.k8s.io/v1alpha1"
    juju ssh kubernetes-master/0 -- 'sudo service snap.kube-apiserver.daemon restart'
    sleep 120 
    juju ssh kubernetes-master/0 -- '/snap/bin/kubectl create clusterrolebinding root-cluster-admin-binding --clusterrole=cluster-admin --user=admin && /snap/bin/kubectl create clusterrolebinding kubelet-node-binding --clusterrole=system:node --user=kubelet'

    echo "obtaining istio"
    curl -L https://git.io/getLatestIstio | sh -
    export ISTIODIR=`ls | grep istio`
    echo "using istiodir $ISTIODIR"
    cd $ISTIODIR
    export PATH=$ISTIODIR/bin:$PATH
    echo "installing istio"
    kubectl apply -f install/kubernetes/istio-auth.yaml 
    kubectl apply -f install/kubernetes/istio-initializer.yaml 
   
    echo "allow traffic to worker" 
    juju expose kubernetes-worker

  EOT
 
  config.vm.provision :shell, privileged: false, run: "always", :inline => <<-EOT
    echo "configuring port forwarding.."

    juju show-status --format yaml kubernetes-worker/0 | grep public-address | head -n1 | cut -d':' -f2 > /tmp/WORKERIP
    echo "Got worker ip of.."
    cat /tmp/WORKERIP

    juju show-status --format yaml kubernetes-master/0 | grep public-address | head -n1 | cut -d':' -f2 > /tmp/MASTERIP
    echo "Got master ip of.."
    cat /tmp/MASTERIP

    cp /home/ubuntu/.kube/config.conjureup /vagrant/kubeconfig
    echo "Using config file.. kubeconfig"

    echo "Fixing config file." 

    ip address show enp0s8 | awk '$1=="inet"{print$2}' | cut -d'/' -f1 > /tmp/EXTERNALIP
    echo "External IP is.."
    cat /tmp/EXTERNALIP
    export EXTERNALIP=`cat /tmp/EXTERNALIP`

    echo "..adding insecure tls to config"
    cat /vagrant/kubeconfig | sed -e 's/certificate-authority-data.*$/insecure-skip-tls-verify: true/' > /vagrant/kubeconfig2 
    echo "..updating external ip in config"
    cat /vagrant/kubeconfig2 | sed -e 's#server:.*$#server: https://'$EXTERNALIP':6443#' > /vagrant/kubeconfig
    rm /vagrant/kubeconfig2

    juju config kubernetes-worker --reset docker-opts docker-opts="--insecure-registry $EXTERNALIP:5000"

    #juju run-action kubernetes-worker/0 registry domain=$EXTERNALIP ingress=true
    #for when we secure docker ;p
    #htpasswd="$(base64 -w0 htpasswd)" htpasswd-plain="$(base64 -w0 htpasswd-plain)" tlscert="$(base64 -w0 registry.crt)" tlkkey="$(base64 -w0 registry.key)"

  EOT

  config.vm.provision :shell, run: "always", :inline => <<-EOT
    export WORKERIP=`cat /tmp/WORKERIP`
    export MASTERIP=`cat /tmp/MASTERIP`
    export EXTERNALIP=`cat /tmp/EXTERNALIP`

    echo "Updating docker to listen on external address "$EXTERNALIP":5000"
    sed -i.bak 's#^ExecStart=.*$#ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2375 --insecure-registry '$EXTERNALIP':5000#' /lib/systemd/system/docker.service 
    systemctl daemon-reload
    service docker restart
  EOT


  config.vm.provision :shell, :inline => <<-EOT
    echo "Launching docker registry"
    docker run -d -p5000:5000 --restart=always --name registry registry:2
  EOT


  config.vm.provision :shell, run: "always", :inline => <<-EOT
    export WORKERIP=`cat /tmp/WORKERIP`
    export MASTERIP=`cat /tmp/MASTERIP`
    export EXTERNALIP=`cat /tmp/EXTERNALIP`

    echo "Adding port fwd via iptables to worker node"
    echo "...http,https"
    iptables -t nat -A PREROUTING -i enp0s8 -p tcp -m multiport --dport 80,443 -j DNAT --to-destination $WORKERIP
    echo "...nodeport range"
    iptables -t nat -A PREROUTING -i enp0s8 -p tcp --dport 30000:32767 -j DNAT --to-destination $WORKERIP
    echo "adding port fwd master port (to master node)"
    iptables -t nat -A PREROUTING -i enp0s8 -p tcp --dport 6443 -j DNAT --to-destination $MASTERIP

    echo "=== KUBERNETES ===="
    echo " kubectl --kubeconfig=./kubeconfig cluster-info"
    echo " kube master is at https://"$EXTERNALIP":6443" 
    echo " kube dashboard is at https://"$EXTERNALIP":6443/api/v1/namespaces/kube-system/services/kubernetes-dashboard/proxy"
    echo "  user id is "
    grep username /vagrant/kubeconfig
    echo "  password is "
    grep password /vagrant/kubeconfig
    echo "=== DOCKER ====" 
    echo "docker is at "$EXTERNALIP":2375"
    echo " export DOCKER_HOST="$EXTERNALIP
    echo "docker registry is at "$EXTERNALIP":5000"
    echo " docker tag your-image:tagname "$EXTERNALIP":5000/your-image:tagname"
    echo " docker push "$EXTERNALIP":5000/your-image:tagname"
    echo "Refer to image from kube yaml as "$EXTERNALIP":5000/your-image:tagname"
  EOT
end
