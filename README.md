# devops
Hi, without 5 minutes, DevOps. <br />
I am using linux academy. And I am writing privateIp on all conf.

  	master ip = 172.31.19.86
  	node ip = 172.31.103.99

######
## Step 1: Create a Repo on all Host i.e. Master, Node1(Minion1)
###
vim /etc/yum.repos.d/virt7-docker-common-release.repo

		[virt7-docker-common-release]
		name=virt7-docker-common-release
		baseurl=http://cbs.centos.org/repos/virt7-docker-common-release/x86_64/os/
		gpgcheck=0
###
######

######
## Step 2: Installing Kubernetes, etcd and flannel on all Host i.e. Master, Node1(Minion1)
###
		yum -y install --enablerepo=virt7-docker-common-release kubernetes etcd flannel
###
######

######
## Step 3: Next Configure Kubernetes Components.
Kubernetes Common Configuration on all Host i.e. Master, Node1(Minion1)
Let’s get started with the common configuration for Kubernetes cluster. This configuration should be done on all the host i.e. Master and Minions. 
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
All the configuration data of Kubernetes is stored in etcd. To increase security, etcd can be bind to the private IP address of the master node. Now, etcd endpoint can only be accessed from the private Subnet.<p>

### API Server Configuration (On Master) <br />
API Server handles the REST operations and acts as a front-end to the cluster’s shared state. API Server Configuration is stored at /etc/kubernetes/apiserver.<p>
Kubernetes uses certificates to authenticate API request. Before configuring API server, we need to generate certificates that can be used for authentication. <p> Kubernetes provides ready made scripts for generating these certificates which can be found <a href="https://github.com/kv020devops/devops/blob/master/make-ca-cert.sh">here</a>.
Download this script and update line number 30 in the file.

	cert_group=${CERT_GROUP:-kube}

###
Now, run the script with following parameters to create certificates:


	bash make-ca-cert.sh "!!!YOURPRIVATIP"  "IP:!!!YOURPRIVATIP,IP:10.254.0.1,DNS:kubernetes,DNS:kubernetes.default,DNS:kubernetes.default.svc,DNS:kubernetes.default.svc.cluster.local"

Now, we can configure API server: 
vi /etc/kubernetes/apiserver

	KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"
	KUBE_API_PORT="--port=8080"
	KUBE_ETCD_SERVERS="--etcd-servers=http://YOURPRIVATIPMASTER:2379"
	KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
	KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"
	KUBE_API_ARGS="--client-ca-file=/srv/kubernetes/ca.crt --tls-cert-file=/srv/kubernetes/server.cert --tls-private-key-file=/srv/kubernetes/server.key"
	
Controller Manager Configuration (On Master) <br />
vi /etc/kubernetes/controller-manager

	KUBE_CONTROLLER_MANAGER_ARGS="--root-ca-file=/srv/kubernetes/ca.crt --service-account-private-key-file=/srv/kubernetes/server.key"

### Kubelet Configuration (On Minions) <br />
Kubelet is a node/minion agent that runs pods and make sure that it is healthy. It also communicates pod details to Kubernetes Master. Kubelet configuration is stored in /etc/kubernetes/kubelet

[On Minion1] <br />
vi /etc/kubernetes/kubelet

	KUBELET_ADDRESS="--address=0.0.0.0"
	KUBELET_PORT="--port=10250"
	KUBELET_HOSTNAME="--hostname-override=YourPrivatIPNode"
	KUBELET_API_SERVER="--api-servers=http://YourPrivatIPMaster:8080"
	KUBELET_ARGS="--cluster-dns=10.254.3.100 --cluster-domain=cluster.local"
####
Before Configuring Flannel for Kubernetes cluster, we need to create network configuration for Flannel in etcd.<br />
So start the etcd node on the master using the following command:

	systemctl start etcd
	
Create a new key in etcd to store Flannel configuration using the following command:

	etcdctl mkdir /kube-centos/network
	
Next, we need to define the network configuration for Flannel:

	etcdctl mk /kube-centos/network/config "{ \"Network\": \"182.30.0.0/16\", \"SubnetLen\": 24, \"Backend\": { \"Type\": \"vxlan\" } }"
	

The above command allocates the 182.30.0.0/16 subnet to the Flannel network. A flannel subnet of CIDR 24 is allocated to each server in Kubernetes cluster.

### Flannel Configuration (On All Nodes)
Kubernetes uses Flannel to build an overlay network for inter-pod communication. Flannel configuration is stored in /etc/sysconfig/flanneld <br />
vi /etc/sysconfig/flanneld

	FLANNEL_ETCD_ENDPOINTS="http://YourPrivateIpMaster:2379"
	FLANNEL_ETCD_PREFIX="/kube-centos/network"
	
## Step 4: Start services on Master and Minion
Master

	for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler kube-proxy docker flanneld; do 
    	systemctl restart $SERVICES; 
    	systemctl enable $SERVICES; 
    	systemctl status $SERVICES; 
	done
	
Node

	for SERVICES in kube-proxy kubelet flanneld docker; do 
    	systemctl restart $SERVICES; 
    	systemctl enable $SERVICES; 
    	systemctl status $SERVICES; 
	done

### Step 5: Check Status of all Services
Make sure etcd, kube-apiserver, kube-controller-manager, kube-scheduler, and flanneld is running on Master and kube-proxy, kubelet, flanneld, and docker is running on the slave. <br />

#### Deploying Addons in Kubernetes
#### Configuring DNS for Kubernetes Cluster

You can download DNS Replication Controller and Service YAML from <a href="https://github.com/kv020devops/devops/tree/master/DNS">my repository</a>. You can also download the latest version of DNS from official Kubernetes repository (kubernetes/cluster/addons/dns).
Next, use the following command to create a replication controller and service: 

	kubectl create -f DNS/skydns-rc.yaml
	kubectl create -f DNS/skydns-svc.yaml
	
Now, configure Kubelet in !!!!!!!all Minion to resolve all DNS queries from our local DNS service.

	vim /etc/kubernetes/kubelet
	# Add your own!
	KUBELET_ARGS="--cluster-dns=10.254.3.100 --cluster-domain=cluster.local"
	
Restart kubelet on all Minions to load the new kubelet configuration.

	systemctl restart kubelet
	
### Configuring Dashboard for Kubernetes Cluster
Kubernetes Dashboard is also deployed as a Pod in the cluster. You can download Dashboard Deployment and Service YAML from <a href="https://github.com/kv020devops/devops/tree/master/Dashboard">my repository</a>. You can also download the latest version of Dashboard from official Kubernetes repository (kubernetes/cluster/addons/dashboard) <br />

After downloading YAML, run the following commands from the master:

	kubectl create -f Dashboard/dashboard-controller.yaml
	kubectl create -f Dashboard/dashboard-service.yaml
	
Now you can access Kubernetes Dashboard on your browser.
Open http://master_public_ip:8080/ui on your browser.


######
