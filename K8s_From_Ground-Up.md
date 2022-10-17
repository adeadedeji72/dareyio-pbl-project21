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

**Install CFSSL and CFSSLJSON**

~~~
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssl \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssljson
chmod +x cfssl cfssljson
sudo mv cfssl cfssljson /usr/local/bin/
~~~

### **AWS CLOUD RESOURCES FOR KUBERNETES CLUSTER** ###
As we already know, we need some machines to run the control plane and the worker nodes. In this section, you will provision EC2 Instances required to run your K8s cluster. You can use Terraform for this. But it is highly recommended to start out first with manual provisioning using awscli and have thorough knowledge about the whole setup. After that, you can destroy the entire project and start all over again using Terraform. This manual approach will solidify your skills and give you the opportunity to face more challenges.

#### **Step 1 – Configure Network Infrastructure** ####
Virtual Private Cloud – VPC

1. Create a directory named k8s-cluster-from-ground-up

1. Create a VPC and store the ID as a variable:
~~~
VPC_ID=$(aws ec2 create-vpc \
--cidr-block 172.31.0.0/16 \
--output text --query 'Vpc.VpcId'
)
~~~

3. Tag the VPC so that it is named:
~~~
NAME=k8s-cluster-from-ground-up
aws ec2 create-tags \
  --resources ${VPC_ID} \
  --tags Key=Name,Value=${NAME}
~~~

4. Enable DNS support for your VPC:
~~~
aws ec2 modify-vpc-attribute \
--vpc-id ${VPC_ID} \
--enable-dns-support '{"Value": true}'
~~~

5. Enable DNS support for hostnames:
~~~
aws ec2 modify-vpc-attribute \
--vpc-id ${VPC_ID} \
--enable-dns-hostnames '{"Value": true}'
~~~

![](awscli_vpcc_config.jpg)

**AWS Region**

6. Set the required region
~~~
AWS_REGION=eu-central-1
~~~
**Dynamic Host Configuration Protocol – DHCP**

7. Configure DHCP Options Set:

Dynamic Host Configuration Protocol (DHCP) is a network management protocol used on Internet Protocol networks for automatically assigning IP addresses and other communication parameters to devices connected to the network using a client–server architecture.

~~~
DHCP_OPTION_SET_ID=$(aws ec2 create-dhcp-options \
  --dhcp-configuration \
    "Key=domain-name,Values=$AWS_REGION.compute.internal" \
    "Key=domain-name-servers,Values=AmazonProvidedDNS" \
  --output text --query 'DhcpOptions.DhcpOptionsId')
~~~

8. Tag the DHCP Option set:
aws ec2 create-tags \
  --resources ${DHCP_OPTION_SET_ID} \
  --tags Key=Name,Value=${NAME}

![](awscli_domain_config.jpg)

9. Associate the DHCP Option set with the VPC:
~~~
aws ec2 associate-dhcp-options \
  --dhcp-options-id ${DHCP_OPTION_SET_ID} \
  --vpc-id ${VPC_ID}
~~~

**Subnet**

10. Create the Subnet:
~~~
SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 172.31.0.0/24 \
  --output text --query 'Subnet.SubnetId')
aws ec2 create-tags \
  --resources ${SUBNET_ID} \
  --tags Key=Name,Value=${NAME}
~~~

**Internet Gateway – IGW**

11. Create the Internet Gateway and attach it to the VPC:
~~~
INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway \
  --output text --query 'InternetGateway.InternetGatewayId')
aws ec2 create-tags \
  --resources ${INTERNET_GATEWAY_ID} \
  --tags Key=Name,Value=${NAME}
aws ec2 attach-internet-gateway \
  --internet-gateway-id ${INTERNET_GATEWAY_ID} \
  --vpc-id ${VPC_ID}
~~~

**Route tables**

12. Create route tables, associate the route table to subnet, and create a route to allow external traffic to the Internet through the Internet Gateway:
~~~
ROUTE_TABLE_ID=$(aws ec2 create-route-table \
  --vpc-id ${VPC_ID} \
  --output text --query 'RouteTable.RouteTableId')
aws ec2 create-tags \
  --resources ${ROUTE_TABLE_ID} \
  --tags Key=Name,Value=${NAME}
aws ec2 associate-route-table \
  --route-table-id ${ROUTE_TABLE_ID} \
  --subnet-id ${SUBNET_ID}
aws ec2 create-route \
  --route-table-id ${ROUTE_TABLE_ID} \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id ${INTERNET_GATEWAY_ID}
~~~
