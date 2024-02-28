# K8S-kops-setup
r# Setup Kubernetes (K8s) Cluster on AWS


1. Create Ubuntu EC2 instance
1. install AWSCLI
   ```sh
    curl https://s3.amazonaws.com/aws-cli/awscli-bundle.zip -o awscli-bundle.zip
    sudo apt update
    sudo apt install unzip python
    unzip 
    #sudo apt-get install unzip - if you dont have unzip in your system
    ./awscli-bundle/install -i /usr/local/aws -b /usr/local/bin/aws
    ```

1. Install kubectl on ubuntu instance
   ```sh
   curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
   # download a specific version, replace the $(curl -L -s https://dl.k8s.io/release/stable.txt) portion of the command with the specific version.
   # curl -LO https://dl.k8s.io/release/v1.29.2/bin/linux/amd64/kubectl
   # curl -LO https://dl.k8s.io/release/v1.29.2/bin/linux/arm64/kubectl
   For example, to download version 1.29.2 on Linux x86-64, type:
    chmod +x ./kubectl
    sudo mv ./kubectl /usr/local/bin/kubectl
   ```

1. Install kops on ubuntu instance
   ```sh
    curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
   #curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl
    chmod +x kops-linux-amd64
    sudo mv kops-linux-amd64 /usr/local/bin/kops
    ```
1. Create an IAM user/role  with Route53, EC2, IAM,VPC and S3 full access

1. Attach IAM role to ubuntu instance
   ```sh
   # Note: If you create IAM user with programmatic access then provide Access keys. Otherwise region information is enough
   aws configure
    ```

1. Create a Route53 private hosted zone (you can create Public hosted zone if you have a domain)
   ```sh
   Routeh53 --> hosted zones --> created hosted zone  
   Domain Name: valaxy.net
   Type: Private hosted zone for Amzon VPC
   ```
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
                                     
1. create an S3 bucket
   ```sh
    aws s3 mb s3://demo.k8s.valaxy.net
# Create an Amazon S3 bucket for the Kubernetes state store
  Kops needs a “state store” to store configuration information of the cluster.  For example, how many nodes, instance type of each node, and Kubernetes version.
  # This post uses the bucket name kubernetes-aws-io. Bucket names must be unique; you have to use a different name. Create an S3 bucket:
  # aws s3api create-bucket --bucket kubernetes-aws-io 
  # aws s3api create-bucket --bucket your-kops-state-store --region us-east-1
I strongly recommend versioning this bucket in case you ever need to revert or recover a previous version of the cluster. This can be enabled using the AWS CLI as well:
# aws s3api put-bucket-versioning --bucket your-kops-state-store --versioning-configuration Status=Enabled


   ```
1. Expose environment variable:
   ```sh
    export KOPS_STATE_STORE=s3://demo.k8s.valaxy.net
   ```
# export KOPS_CLUSTER_NAME=your-k8s-cluster-name
# For convenience, you can also define KOPS_STATE_STORE environment variable pointing to the S3 bucket. For example:
# export KOPS_STATE_STORE=s3://kubernetes-aws-io

1. Create sshkeys before creating cluster
   ```sh
    ssh-keygen
   ```

1. Create kubernetes cluster definitions on S3 bucket
   ```sh
   kops create cluster --name=demo.k8s.valaxy.net --cloud=aws --zones=ap-south-1b  --dns-zone=valaxy.net --dns private

# The Kops CLI can be used to create a highly available cluster, with multiple master nodes spread across multiple Availability Zones. Workers can be spread across multiple zones as well. Some of the tasks that happen behind the scene during cluster creation are:

                                 Provisioning EC2 instances
                                 Setting up AWS resources such as networks, Auto Scaling groups, IAM users, and security groups
                                 Installing Kubernetes.
Start the Kubernetes cluster using the following command:
   kops create cluster --name cluster.kubernetes-aws.io --zones us-west-2a --state s3://kubernetes-aws-io --node-count=2 --yes

     # Create a cluster in AWS in a single zone. 
  kops create cluster --name=k8s-cluster.example.com \
  --state=s3://my-state-store \
  --zones=us-east-1a \
  --node-count=2

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
   
    ```
# kops create cluster --node-count=2 --node-size=t2.

Suggestions:
 * validate cluster: kops validate cluster --wait 10m
 * list nodes: kubectl get nodes --show-labels
 * ssh to a control-plane node: ssh -i ~/.ssh/id_rsa ubuntu@api.kops-cluster.kops.com
 * the ubuntu user is specific to Ubuntu. If not using Ubuntu please use the appropriate user based on your OS.
 * read about installing addons at: https://kops.sigs.k8s.io/addons.

1. If you wish to update the cluster worker node sizes use below command 
   ```sh 
   kops edit ig --name=CHANGE_TO_CLUSTER_NAME nodes
   ```

1. Create kubernetes cluser
    ```sh
    kops update cluster demo.k8s.valaxy.net --yes
    ```

1. Validate your cluster
     ```sh
      kops validate cluster
    ```

1. To list nodes
   ```sh
   kubectl get nodes
   ```

1. To delete cluster
    ```sh
     kops delete cluster demo.k8s.valaxy.net --yes
    ```
   
#### Deploying Nginx pods on Kubernetes
1. Deploying Nginx Container
    ```sh
    kubectl create deploy sample-nginx --image=nginx --replicas=2 --port=80
    # kubectl deploy simple-devops-project --image=yankils/simple-devops-image --replicas=2 --port=8080
    kubectl get all
    kubectl get pod
   ```

1. Expose the deployment as service. This will create an ELB in front of those 2 containers and allow us to publicly access them.
   ```sh
   kubectl expose deployment sample-nginx --port=80 --type=LoadBalancer
   # kubectl expose deployment simple-devops-project --port=8080 --type=LoadBalancer
   kubectl get services -o wide
   ```
