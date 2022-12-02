# A complete guid to setup Splunk Clustering using Ansible & Terraform!


### What is Terraform?

Terraform is a free and open-source infrastructure as code (IAC) that can help to automate the deployment, configuration, and management of the remote servers. Terraform can manage both existing service providers and custom in-house solutions.

![5](https://github.com/DhruvinSoni30/Jenkins-Terraform-Docker/blob/main/5.png?raw=true)

### What is Ansible?

Ansible is an open-source software provisioning, configuration management, and deployment tool. It runs on many Unix-like systems and can configure both Unix-like systems as well as Microsoft Windows. Ansible uses SSH protocol in order to configure the remote servers. Ansible follows the push-based mechanism to configure the remote servers.

![3](https://github.com/DhruvinSoni30/Jenkins-Ansible-Demo/blob/main/3.png?raw=true)

### Prerequisites:

* A server that has Terraform & Ansible installed in it & AWS CLI also!
* Basic understanding of Terraform & Ansible
* Basic understanding of AWS
* AWS Access Key & Secret Key

### For this tutorial we will use terraform to spin up various instances and install splunk in it & ansible to configure the clustering!

> We will use separate variables file for storing all the variables. So, at the end I will discuss that file also.

**Step 1:- Configure AWS credentials**

* The server in which you have installed terraform run below command to configure AWS credentials

  ```
  aws configure
  ```
  The above command will ask you to enter the Access key, secret key, output format and default region, please provide all the details

**Step 2:- Create Provider block**
  
  ```
  provider "aws" {
    region     = "${var.aws_region}"
  }
  ```

* Provider block is used to configure and download plugins for the various cloud provider
* We are using variable for region

**Step 3:- Creating AWS VPC**

  ```
  resource "aws_vpc" "demovpc" {
    cidr_block       = "${var.vpc_cidr}"
    instance_tenancy = "default"
  tags = {
    Name = "Demo VPC"
  }
  }
  ```
  
* aws_vpc is the VPC module for AWS
* We are using variable for cidr_block
* demovpc is the logical name of VPC resource
* I have set instance_tenancy to default
* Demo VPC is the tag of the VPC

**Step 4:- Creating Internet Gateway**

  ```
  resource "aws_internet_gateway" "demogateway" {
    vpc_id = "${aws_vpc.demovpc.id}"
  }
  ```
  
* aws_internet_gateway is the Internet Gateway module for AWS
* demogateway is the logical name of Internet Gateway
* We are using the newly created VPC’s ID and attaching IGW to that VPC in this code vpc_id = “${aws_vpc.demovpc.id}"

**Step 5:- Updating the AWS Route Table**

  ```
  resource "aws_route" "internet_access" {
    route_table_id         = "${aws_vpc.demovpc.main_route_table_id}"
    destination_cidr_block = "0.0.0.0/0"
    gateway_id             = "${aws_internet_gateway.demogateway.id}"
  }
  ```

* aws_route is the Route Table module for AWS
* internet_access is the logical name of the Route table
* We are using the ID of the main route table in this block aws_vpc.demovpc.main_route_table_id
* We are adding the destination CIDR block as 0.0.0.0/0 for internet access.
* We are attaching the updated route table to the newly created IGW in this block aws_internet_gateway.demogateway.id


**Step 6:- Creating AWS Subnet**

  ```
  resource "aws_subnet" "demosubnet" {
    vpc_id                  = "${aws_vpc.demovpc.id}"
    cidr_block             = "${var.subnet_cidr}"
    map_public_ip_on_launch = true
  tags = {
    Name = "Demo subnet"
  }
  }
  ```

* aws_subnet is the subnet module for AWS
* demosubnet is the logical name of the subnet
* We are attaching the subnet to the newly created VPC in this block aws_vpc.demovpc.id
* We are using variable for cidr_block
* map_public_ip_on_launch will attach public IP to EC2 instances that are going to launch in this subnet
* Demo subnet is the tag

**Step 7:- Creating Inbound AWS Security Group**

  ```
  resource "aws_security_group" "demosg" {
    name        = "Demo Security Group"
    description = "Demo Module"
    vpc_id      = "${aws_vpc.demovpc.id}"
  # Inbound Rules
  # HTTP access from anywhere
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  # HTTPS access from anywhere
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  # SSH access from anywhere
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ```

* aws_security_group is the Security Group module for AWS
* demosg is the logical name of the subnet
* We are creating this SG in the newly created VPC in this code aws_vpc.demovpc.id
* We are creating Inbound rules for 80, 443 & 22 ports and allowing access for all the IPs

**Step 8:- Creating Outbound AWS Security Group**
  
  ```
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  }
  ```

* We are opening outbound connection for all the IPs from anywhere

**Step 9:- Creating key pair for AWS EC2 Instance**

  ```
  resource "aws_key_pair" "demokey" {
    key_name   = "${var.key_name}"
    public_key = "${file(var.public_key)}"
  }
  ```
  
* aws_key_pair is the key pair module for AWS
* demokey is the logical name for key pair
* We are using the file function to take the value from the public_key variable for the key pair
* So, basically, it will take the value of the SSH key pair from the tests.pub file.


  
