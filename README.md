# devops
Hi, without 5 minutes, DevOps. <br />
I am using linux academy. And I am writing privateIp on all conf.

  	master ip = 172.31.19.86
  	node ip = 172.31.103.99

######
Step 1: Create a Repo on all Host i.e. Master, Node1(Minion1)
###
vim /etc/yum.repos.d/virt7-docker-common-release.repo

		[virt7-docker-common-release]
		name=virt7-docker-common-release
		baseurl=http://cbs.centos.org/repos/virt7-docker-common-release/x86_64/os/
		gpgcheck=0
###
######

######
Step 2: Installing Kubernetes, etcd and flannel on all Host i.e. Master, Node1(Minion1)
###
		yum -y install --enablerepo=virt7-docker-common-release kubernetes etcd flannel
###
######

######
Step 3: Next Configure Kubernetes Components.
Kubernetes Common Configuration on all Host i.e. Master, Node1(Minion1)
###
vi /etc/kubernetes/config
		 
		KUBE_LOGTOSTDERR="--logtostderr=true"

		# journal message level, 0 is debug
		KUBE_LOG_LEVEL="--v=0"

		# Should this cluster be allowed to run privileged docker containers
		KUBE_ALLOW_PRIV="--allow-privileged=false"

		# How the controller-manager, scheduler, and proxy find the apiserver
		KUBE_MASTER="--master=http://172.31.19.86:8080"
		
###
###
ETCD Configuration (On Master!!!) <br />
vim /etc/etcd/etcd.conf 
			
			ETCD_NAME=default
			ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
			TCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
			#[cluster]
			ETCD_ADVERTISE_CLIENT_URLS="http://0.0.0.0:2379"
			
###
API Server Configuration (On Master)
Download this script(<a href="#">make-ca-cert.sh</a>) and update line number 30 in the file.

######


