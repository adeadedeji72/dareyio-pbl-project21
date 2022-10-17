## K8s From-Ground-Up ##

 we will implement Kubernetes architecture without any automated helpers. We will install each and every component manually from scratch and learn how to make them work together – we call this approach "K8s From-Ground-Up".

Tools required
- VM: AWS EC2
- OS: Ubuntu 20.04 lts+
- Docker Engine
- kubectl console utility
- *cfssl* and *cfssljson* utilities
- Kubernetes cluster

You will create 3 EC2 Instances, and in the end, we will have the following parts of the cluster properly configured:

- One Kubernetes Master
- Two Kubernetes Worker Nodes
- Configured SSL/TLS certificates for Kubernetes components to communicate securely
- Configured Node Network
- Configured Pod Network

### **STEP 0-INSTALL CLIENT TOOLS BEFORE BOOTSTRAPPING THE CLUSTER** ###

- **awscli** – is a unified tool to manage your AWS services
- **kubectl** – this command line utility will be your main control tool to manage your K8s cluster. You will use this tool so many times, so you will be able to type
‘kubetcl’ on your keyboard with a speed of light. You can always make a shortcut (alias) to just one character ‘k’. Also, add this extremely useful official kubectl Cheat Sheet to your bookmarks, it has examples of the most used ‘kubectl’ commands.
- **cfssl** – an open source toolkit for everything TLS/SSL from Cloudflare
- **cfssljson** – a program, which takes the JSON output from the cfssl and writes certificates, keys, CSRs, and bundles to disk.


**Install and configure AWS CLI**
- Configure AWS CLI to access all AWS services used, for this you need to have a user with programmatic access keys configured in AWS Identity and Access Management (IAM), Generate access keys and store them in a safe place
- Install awscli on the workstation
~~~
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
./aws/install -i /usr/local/aws-cli -b /usr/local/bin
~~~
Verify your installation with

~~~
aws --version 
~~~

To configure your AWS CLI – run your shell commands:
~~~
aws configure --profile %your_username%
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-west-1
Default output format [None]: json
~~~

Test AWS cli with:

~~~
aws ec2 describe-vpcs
~~~
You should see the description of your vpcs in json format.

**Install kubectl**
Kubernetes cluster has a Web API that can receive HTTP/HTTPS requests, but it is quite cumbersome to curl an API each and every time you need to send some command, so kubectl command tool was developed to ease a K8s administrator’s life. We will use kubectl to ease our adminstration tasks.

Download the binary
~~~
wget https://storage.googleapis.com/kubernetes-release/release/v1.25.3/bin/linux/amd64/kubectl
~~~
Make it executable
~~~
chmod +x kubectl
~~~
Move to the Bin directory
~~~
sudo mv kubectl /usr/local/bin/
~~~

Verify that kubectl version 1.25.3 or higher is installed:
~~~
kubectl version --client
~~~
