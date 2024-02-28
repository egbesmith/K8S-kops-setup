# kops
The easiest way to get a production grade Kubernetes cluster up and running.
# Kubernetes and Kops overview
Kubernetes is an open source, container orchestration platform. Applications packaged as Docker images can be easily deployed, scaled, and managed in a Kubernetes cluster. 
# Some of the key features of Kubernetes are:

1. Self-healing
Failed containers are restarted to ensure that the desired state of the application is maintained. If a node in the cluster dies, then the containers are rescheduled on a different node. Containers that do not respond to application-defined health check are terminated, and thus rescheduled.
2. Horizontal scaling
Number of containers can be easily scaled up and down automatically based upon CPU utilization, or manually using a command.
3. Service discovery and load balancing
Multiple containers can be grouped together discoverable using a DNS name. The service can be load balanced with integration to the native LB provided by the cloud provider.
4. Application upgrades and rollbacks
Applications can be upgraded to a newer version without an impact to the existing one. If something goes wrong, Kubernetes rolls back the change.
Kops, short for Kubernetes Operations, is a set of tools for installing, operating, and deleting Kubernetes clusters in the cloud. A rolling upgrade of an older version of Kubernetes to a new version can also be performed. It also manages the cluster add-ons. After the cluster is created, the usual kubectl CLI can be used to manage resources in the cluster.

# what is kops?

kops will not only help you create, destroy, upgrade and maintain production-grade, highly available, Kubernetes cluster, but it will also provision the necessary cloud infrastructure. kops can be use in any cloud provider: AWS (Amazon Web Services) and GCE (Google Cloud Platform) are currently officially supported, with DigitalOcean, Hetzner and OpenStack in beta support, and Azure in alpha# K8S-kops-setup

# The different download options for kops are explained at github.com/kubernetes/kops#installing. On MacOS, the easiest way to install kops is using the brew package manager.
There is no need to download the Kubernetes binary distribution for creating a cluster using kops. However, you do need to download the kops CLI. It then takes care of downloading the right Kubernetes binary in the cloud, and provisions the cluster.

# general procedure for kop-setup
1. Create Ubuntu EC2 instance
2. Install AWSCLI
3. Install kubectl on ubuntu instance
4. Install kops on ubuntu instance
5. Create an IAM user/role  with Route53, EC2, IAM,VPC and S3 full access
6. Attach IAM role to ubuntu instance
7. Create a Route53 private hosted zone (you can create Public hosted zone if you have a domain)
   # DNS configuration
 As of Kops 1.6.1, a top-level domain or a subdomain is required to create the cluster. This domain allows the worker nodes to discover the master and the master to 
 discover all the etcd servers. This is also needed for kubectl to be able to talk directly with the master.
# This domain may be registered with AWS, in which case a Route 53 hosted zone is created for you. Alternatively, this domain may be at a different registrar. In this case, create a Route 53 hosted zone. Specify the name server (NS) records from the created zone as NS records with the domain registrar.
# Generate a Route 53 hosted zone using the AWS CLI. Download jq to run this command:

                                    ID=$(uuidgen) && \
                                    aws route53 create-hosted-zone \
                                    --name cluster.kubernetes-aws.io \
                                    --caller-reference $ID \
                                    | jq .DelegationSet.NameServers
                                     
8. create an S3 bucket
Create an Amazon S3 bucket for the Kubernetes state store. Kops needs a “state store” to store configuration information of the cluster.  For example, how many nodes, instance type of each node, and Kubernetes version.

# I strongly recommend versioning this bucket in case you ever need to revert or recover a previous version of the cluster. This can be enabled using the AWS CLI as well:
9. Expose environment variable:
# For convenience, you can also define KOPS_STATE_STORE environment variable pointing to the S3 bucket. For example: export KOPS_STATE_STORE=s3://kubernetes-aws-io

10. Create sshkeys before creating cluster
   
11. Create kubernetes cluster definitions on S3 bucket
# The Kops CLI can be used to create a highly available cluster, with multiple master nodes spread across multiple Availability Zones. Workers can be spread across multiple zones as well. Some of the tasks that happen behind the scene during cluster creation are:

                                 Provisioning EC2 instances
                                 Setting up AWS resources such as networks, Auto Scaling groups, IAM users, and security groups
                                 Installing Kubernetes.
Start the Kubernetes cluster using the following command:

kops create cluster --name cluster.kubernetes-aws.io --zones us-west-2a --state s3://kubernetes-aws-io --node-count=2 --yes

# In this command:
--zones
Defines the zones in which the cluster is going to be created. Multiple comma-separated zones can be specified to span the cluster across multiple zones.
--name
Defines the cluster’s name.
--state
Points to the S3 bucket that is the state store.
--yes
Immediately creates the cluster. Otherwise, only the cloud resources are created and the cluster needs to be started explicitly using the command kops update --yes. If the cluster needs to be edited, then the kops edit cluster command can be used.
This starts a single master and two worker node Kubernetes cluster. The master is in an Auto Scaling group and the worker nodes are in a separate group. By default, the master node is m3.medium and the worker node is t2.medium. Master and worker nodes are assigned separate IAM roles as well.
   
Suggestions:
 * validate cluster: kops validate cluster --wait 10m
 * list nodes: kubectl get nodes --show-labels
 * ssh to a control-plane node: ssh -i ~/.ssh/id_rsa ubuntu@api.kops-cluster.kops.com
 * the ubuntu user is specific to Ubuntu. If not using Ubuntu please use the appropriate user based on your OS.
 * read about installing addons at: https://kops.sigs.k8s.io/addons.

#### Deploying Nginx pods on Kubernetes
1. Deploying Nginx Container
2. Expose the deployment as service. This will create an ELB in front of those 2 containers and allow us to publicly access them.
   
