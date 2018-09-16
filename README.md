# Using Terraform to Build AWS EKS



## Prerequisites

* A AWS Cloud9 console and have **AdministratorAccess** Role
* This Lab will use **US WEST (Oregon)**


## Lab tutorial
### Turn Off Cloud9 temporary credentials:
Open Aws Cloud9 Console and in Preferences, find AWS Settings, turn off the AWS managed temporary credentials.

![1.png](/images/1.png)



### Install Terraform

Copy below command to install terraform.

	$ wget https://releases.hashicorp.com/terraform/0.11.8/terraform_0.11.8_linux_amd64.zip
    $ unzip terraform_0.11.8_linux_amd64.zip
    $ rm terraform_0.11.8_linux_amd64.zip
    $ sudo mv terraform /usr/bin

Use below command to verify terraform has been installed.

	$ terrafom
    Usage: terraform [--version] [--help] <command> [args]

	The available commands for execution are listed below.
	The most common, useful commands are shown first, followed by
	less common or more advanced commands. If you're just getting
	started with Terraform, stick with the common commands. For the
	other commands, please read the help and docs before usage.

	Common commands:
    	apply              Builds or changes infrastructure
    	console            Interactive console for Terraform interpolations
	# ...

### Install kubectl

Copy below command to install Kubectl.

	$ curl -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/linux/amd64/kubectl
    $ chmod +x kubectl
    $ sudo mv kubectl /usr/bin
Use below command to verify kubectl has been installed.   

	$ kubectl version --short --client

It should be output:

	Client Version: v1.10.3
	
### Install aws-iam-authenticator for Amazon EKS

Copy below command to install aws-iam-authenticator.

	$ curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/linux/amd64/aws-iam-authenticator
    $ chmod +x aws-iam-authenticator
    $ sudo mv aws-iam-authenticator /usr/bin
 
Use below command to verify aws-iam-authenticator has been installed.

	$ aws-iam-authenticator help
    
### Try to apply Terraform Template

Download the repository.

	$ git clone https://github.com/SypherSu/eks_terraform_sample.git
    $ cd eks_terraform_sample
    
First, open cluster.tf, and find **aws_security_group_rule** **demo-cluster-ingress-workstation-https**, and change the **cidr_blocks** to your Cloud9 Public IP.

Next we need to initiate the directory.

	$ terraform init
    
Plan the terraform template, look what will be created.

	$ terraform plan

Apply the terraform template.

	$ terraform apply
    
When the terraform ask you which region you want to create, please type **us-west-2**.

Next, If terraform check all templates are good, please type **yes** to apply.

Until complete, you need to wait **20** minutes.

When terraform give you two outputs,like below:

	Apply complete! Resources: 25 added, 0 changed, 0 destroyed.

	Outputs:

	config-map-aws-auth = 

	apiVersion: v1
    kind: ConfigMap
    metadata:
      name: aws-auth
      namespace: kube-system
    data:
      mapRoles: |
        - rolearn: <Your role ARN>
          username: system:node:{{EC2PrivateDNSName}}
          groups:
            - system:bootstrappers
            - system:nodes

    kubeconfig = 

    apiVersion: v1
    clusters:
    - cluster:
        server: <Your Endpoint>
        certificate-authority-data: <Your CA>
      name: kubernetes
    contexts:
    - context:
        cluster: kubernetes
        user: aws
      name: aws
    current-context: aws
    kind: Config
    preferences: {}
    users:
    - name: aws
      user:
        exec:
          apiVersion: client.authentication.k8s.io/v1alpha1
          command: aws-iam-authenticator
          args:
            - "token"
            - "-i"
            - "<Your Cluster Name>"

### To create your kubeconfig file for kubectl

Create the default ~/.kube directory if it does not already exist.
	
    $ mkdir ~/.kube
    
Create a new file and copy the kubeconfig code that terrafom output for you into it, it will look like below.
	
    apiVersion: v1
    clusters:
    - cluster:
        server: <endpoint-url>
        certificate-authority-data: <base64-encoded-ca-cert>
      name: kubernetes
    contexts:
    - context:
        cluster: kubernetes
        user: aws
      name: aws
    current-context: aws
    kind: Config
    preferences: {}
    users:
    - name: aws
      user:
        exec:
          apiVersion: client.authentication.k8s.io/v1alpha1
          command: aws-iam-authenticator
          args:
            - "token"
            - "-i"
            - "<cluster-name>"
 
Save the file with your cluster name in the file name. For example, if your cluster name is **devel**, save the file like **config-devel**. And run below command:

	$ sudo mv <Your Config File> ~/.kube

Add that file path to your KUBECONFIG environment variable so that kubectl knows where to look for your cluster configuration.

	$ export KUBECONFIG=$KUBECONFIG:~/.kube/config-<your-cluster-name>

Now try to test your configuration:

	$ kubectl get svc
    
If success, you will get your cluster.

### Configure Amazon EKS Worker Nodes

Create a new file and copy the config-map-aws-auth code that terrafom output for you into it, it will look like below.

	apiVersion: v1
    kind: ConfigMap
    metadata:
      name: aws-auth
      namespace: kube-system
    data:
      mapRoles: |
        - rolearn: <Your role ARN>
          username: system:node:{{EC2PrivateDNSName}}
          groups:
            - system:bootstrappers
            - system:nodes

Save the file name **aws-auth-cm.yaml**.

Apply the configuration. This command may take a few minutes to finish.

	$ kubectl apply -f aws-auth-cm.yaml

Watch the status of your nodes and wait for them to reach the Ready status.

	$ kubectl get nodes --watch
    
### Test

Create the Redis master replication controller.
	
    $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/v1.10.3/examples/guestbook-go/redis-master-controller.json
    
Create the Redis master service.

	kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/v1.10.3/examples/guestbook-go/redis-master-service.json

Create the Redis slave replication controller.
	
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/v1.10.3/examples/guestbook-go/redis-slave-controller.json
    
Create the Redis slave service.

	kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/v1.10.3/examples/guestbook-go/redis-slave-service.json
    
Create the guestbook replication controller.

	kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/v1.10.3/examples/guestbook-go/guestbook-controller.json

Create the guestbook service.

	kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/v1.10.3/examples/guestbook-go/guestbook-service.json

After your external IP address is available, point a web browser to that address at port 3000 to view your guest book. 

For example, http://a7a95c2b9e69711e7b1a3022fdcfdf2e-1985673473.us-west-2.elb.amazonaws.com:3000

**It may take several minutes for DNS to propagate and for your guest book to show up.**

![2.png](/images/2.png)

Clean All Test:

	kubectl delete rc/redis-master rc/redis-slave rc/guestbook svc/redis-master svc/redis-slave svc/guestbook
    
### Clean All Resources:

Use below command to clean all resources that use terraform to create.

	$ terraform destroy









