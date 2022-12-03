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

# Part 1 (Terraform)

> We will use separate variables file for storing all the variables. So, at the end I will discuss that file also.

**Step 1:- Configure AWS credentials**

* The server in which you have installed terraform run below command to configure AWS credentials

  ```
  aws configure
  ```
  The above command will ask you to enter the Access key, secret key, output format and default region, please provide all the details

**Step 2:- Creating Provider block**

* Defining AWS region
  
  ```
  provider "aws" {
    region     = "${var.aws_region}"
  }
  ```

**Step 3:- Creating AWS VPC**

* Defining AWS VPC

  ```
  # Create a VPC to launch
   resource "aws_vpc" "default" {
     cidr_block = "10.0.0.0/16"
  }
  ```

**Step 4:- Creating Internet Gateway**

* Defining AWS IGW

  ```
  resource "aws_internet_gateway" "demogateway" {
    vpc_id = "${aws_vpc.demovpc.id}"
  }
  ```

**Step 5:- Updating the AWS Route Table**

* Updating AWS Route Table

  ```
  resource "aws_route" "internet_access" {
    route_table_id         = "${aws_vpc.demovpc.main_route_table_id}"
    destination_cidr_block = "0.0.0.0/0"
    gateway_id             = "${aws_internet_gateway.demogateway.id}"
  }
  ```

**Step 6:- Creating AWS Subnet**

* Defining AWS Subnet for Cluster Master, Indexers, SearchHeads & Forwarders!

  ```
  # Create a subnet to launch our instances into
  resource "aws_subnet" "default" {
    vpc_id                  = "${aws_vpc.demovpc.id}"
    cidr_block              = "10.0.1.0/24"
    map_public_ip_on_launch = true
  }

  resource "aws_subnet" "forwarder" {
    vpc_id                  = "${aws_vpc.demovpc.id}"
    cidr_block               "10.0.2.0/24"
    map_public_ip_on_launch = true
  }
  ```

**Step 7:- Creating Inbound AWS Security Group for ELB**

* Defining AWS Security Group for ELB

  ```
  # A security group for the ELB so it is accessible via the web
  resource "aws_security_group" "elb" {
    name        = "terraform_example_elb"
    description = "Used in the terraform"
    vpc_id      = "${aws_vpc.demovpc.id}"

    # HTTP access from anywhere
    ingress {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  
    ingress {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  
    # outbound internet access
    egress {
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
  ```

**Step 8:- Creating Inbound AWS Security Group for Instances**
  
* Defining AWS Security Group for Instances

  ```
  # Our default security group to access
  # the instances over SSH and HTTP
  resource "aws_security_group" "default" {
    name        = "Project - Terraform"
    description = "Project - Terraform"
    vpc_id      = "${aws_vpc.demovpc.id}"

  # SSH access from anywhere
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTP access from the VPC
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  ingress {
    from_port   = 8000
    to_port     = 8000
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 8089
    to_port     = 8089
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 4598
    to_port     = 4598
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 2511
    to_port     = 2511
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 9997
    to_port     = 9997
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # outbound internet access
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  }
  ```


**Step 9:- Creating key pair for AWS EC2 Instance**

* Defining AWS Key Pair

  ```
  resource "aws_key_pair" "demokey" {
    key_name   = "${var.key_name}"
    public_key = "${file(var.public_key)}"
  }
  ```
  
**Step 10:- Creating instance for Cluster Master & install splunk in it**

* Defining AWS Instance for Cluster Master

  ```
  #MasterSetup
  resource "aws_instance" "master" {
  # The connection block tells our provisioner how to
  # communicate with the resource (instance)
    connection {
      type = "ssh"
      # The default username for our AMI
      user = "ec2-user"
      private_key = "${file(var.private_key_path)}"
      # The connection will use the local SSH agent for authentication.
    }

    instance_type = "t2.micro"

    # Lookup the correct AMI based on the region we specified
    ami = "${lookup(var.aws_amis, var.aws_region)}"
    tags = {
      Name = "${format("master")}"
      Name = "master"
    }
  
    # Root Block Storage
    root_block_device {
      volume_size = "40"
      volume_type = "standard"
    }
    #EBS Block Storage
    ebs_block_device {
      device_name = "/dev/sdb"
      volume_size = "80"
      volume_type = "standard"
      delete_on_termination = false
    }

    # The name of our SSH keypair we created above.
    key_name = "${aws_key_pair.auth.id}"

    # Our Security group to allow HTTP and SSH access
    vpc_security_group_ids = ["${aws_security_group.default.id}"]

    # We're going to launch into the same subnet as our ELB. In a production
    # environment it's more common to have a separate private subnet for
    # backend instances.
    subnet_id = "${aws_subnet.default.id}"

    # We run a remote provisioner on the instance after creating it.
    # In this case, we just install nginx and start it. By default,
    # this should be on port 80
    provisioner "remote-exec" {
      inline = [
        "wget https://download.splunk.com/products/splunk/releases/8.1.0/linux/splunk-8.1.0-f57c09e87251-Linux-x86_64.tgz",
        "tar xvzf splunk-8.1.0-f57c09e87251-Linux-x86_64.tgz",
        "cd splunk/bin/",
        "./splunk start --accept-license --answer-yes --no-prompt --seed-passwd 66546654",

      ]
    }
  }
  ```

**Step 11:- Creating Instances for Indexers & install splunk in it**

* Defining AWS Instance for Indexers

  ```
  #IndexerSetup
  resource "aws_instance" "indexer" {
  # The connection block tells our provisioner how to
  # communicate with the resource (instance)
    connection {
      type = "ssh"
      # The default username for our AMI
      user = "ec2-user"
      private_key = "${file(var.private_key_path)}"
      # The connection will use the local SSH agent for authentication.
    }

    instance_type = "t2.micro"

    # Lookup the correct AMI based on the region we specified
    ami = "${lookup(var.aws_amis, var.aws_region)}"
    count = "${var.aws_indexer_count}"
    tags = {
      Name = "${format("indexer%01d",count.index+1)}"
    }
    # The name of our SSH keypair we created above.
    key_name = "${aws_key_pair.auth.id}"

    # Our Security group to allow HTTP and SSH access
    vpc_security_group_ids = ["${aws_security_group.default.id}"]

    # We're going to launch into the same subnet as our ELB. In a production
    # environment it's more common to have a separate private subnet for
    # backend instances.
    subnet_id = "${aws_subnet.default.id}"

    # We run a remote provisioner on the instance after creating it.
    # In this case, we just install nginx and start it. By default,
    # this should be on port 80
    provisioner "remote-exec" {
      inline = [
        "wget https://download.splunk.com/products/splunk/releases/8.1.0/linux/splunk-8.1.0-f57c09e87251-Linux-x86_64.tgz",
        "tar xvzf splunk-8.1.0-f57c09e87251-Linux-x86_64.tgz",
        "cd splunk/bin/",
        "./splunk start --accept-license --answer-yes --no-prompt --seed-passwd 66546654",

      ]
    }
  }
  ```

**Step 12:- Creating instances for SearchHeads & install Splunk it**

* Defining AWS Instance for SearchHeads

  ```
  #SearchHeadSetup
  resource "aws_instance" "search" {
  # The connection block tells our provisioner how to
  # communicate with the resource (instance)
    connection {
      type = "ssh"
      # The default username for our AMI
      user = "ec2-user"
      private_key = "${file(var.private_key_path)}"
      # The connection will use the local SSH agent for authentication.
    }

    instance_type = "t2.micro"

    # Lookup the correct AMI based on the region we specified
    ami = "${lookup(var.aws_amis, var.aws_region)}"
    count = "${var.aws_search_count}"
    tags = {
      Name = "${format("search%01d",count.index+1)}"
    }
  
    # Root Block Storage
    root_block_device {
      volume_size = "40"
      volume_type = "standard"
    }
    #EBS Block Storage
    ebs_block_device {
      device_name = "/dev/sdb"
      volume_size = "80"
      volume_type = "standard"
      delete_on_termination = false
    }

    # The name of our SSH keypair we created above.
    key_name = "${aws_key_pair.auth.id}"

    # Our Security group to allow HTTP and SSH access
    vpc_security_group_ids = ["${aws_security_group.default.id}"]

    # We're going to launch into the same subnet as our ELB. In a production
    # environment it's more common to have a separate private subnet for
    # backend instances.
    subnet_id = "${aws_subnet.default.id}"

    # We run a remote provisioner on the instance after creating it.
    # In this case, we just install nginx and start it. By default,
    # this should be on port 80
    provisioner "remote-exec" {
      inline = [
        "wget https://download.splunk.com/products/splunk/releases/8.1.0/linux/splunk-8.1.0-f57c09e87251-Linux-x86_64.tgz",
        "tar xvzf splunk-8.1.0-f57c09e87251-Linux-x86_64.tgz",
        "cd splunk/bin/",
        "./splunk start --accept-license --answer-yes --no-prompt --seed-passwd 66546654",

      ]
    }
  }
  ```

**Step 13:- Creating instances for Forwarders & install splunk in it**

* Defining AWS Instance for Forwarders

  ```
  #ForwarderSetup
  resource "aws_instance" "forwarder" {
  # The connection block tells our provisioner how to
  # communicate with the resource (instance)
    connection {
      type = "ssh"
      # The default username for our AMI
      user = "ec2-user"
      private_key = "${file(var.private_key_path)}"
      # The connection will use the local SSH agent for authentication.
    }

    instance_type = "t2.micro"

    # Lookup the correct AMI based on the region
    # we specified
    ami = "${lookup(var.aws_amis, var.aws_region)}"
    count = "${var.aws_forwarder_count}"
    tags = {
      Name = "${format("forwarder%01d",count.index+1)}"
    }
    # The name of our SSH keypair we created above.
    key_name = "${aws_key_pair.auth.id}"

    # Our Security group to allow HTTP and SSH access
    vpc_security_group_ids = ["${aws_security_group.default.id}"]

    # We're going to launch into the same subnet as our ELB. In a production
    # environment it's more common to have a separate private subnet for
    # backend instances.
    subnet_id = "${aws_subnet.forwarder.id}"
 
    # We run a remote provisioner on the instance after creating it.
    # In this case, we just install nginx and start it. By default,
    # this should be on port 80
    provisioner "remote-exec" {
      inline = [
        "wget https://download.splunk.com/products/universalforwarder/releases/9.0.2/linux/splunkforwarder-9.0.2-17e00c557dc1-Linux-x86_64.tgz",
        "tar xvzf splunkforwarder-9.0.2-17e00c557dc1-Linux-x86_64.tgz",
        "cd splunkforwarder/bin/",
        "./splunk start --accept-license --answer-yes --no-prompt --seed-passwd 66546654",

      ]
    }
  }
  ```

**Step 14:- Creating variables file**

  ```
  variable "public_key_path" {
  default = "Part1.pub"
  }

  variable "private_key_path" {
    default = "Part1.pem"
  }

  variable "key_name" {
    default     = "instance"
    description = "Desired name of AWS key pair"
  }

  variable "aws_search_count" {
    default = 2
  } 

  variable "aws_indexer_count" {
    default = 2
  }

  variable "aws_forwarder_count" {
    default = 2
  }

  variable "aws_region" {
    description = "AWS region to launch servers."
    default     = "us-east-2"
  }

  # Ubuntu Precise 16.04 LTS (x64)
  variable "aws_amis" {
    default = {
      us-east-2 = "ami-0beaa649c482330f7"
    }
  }
  ```

Run below commands to spin up the various resources of AWS
  ```
  terraform init
  terraform plan
  terraform apply
  ```
 
Above command will create below resources in AWS

![4](https://github.com/DhruvinSoni30/Terraform_Ansible_Splunk_Cluster/blob/main/images/1.png)

# Part 2(Ansible)

**Step 1:- Creating AWS inventory file**

* Create `aws_ec2.yaml` file and add below code in it

  ```
  plugin: aws_ec2
  regions:
    - "us-east-2"
  keyed_groups:
    - key: tags.Name
      prefix: tag_Name
  filters:
    instance-state-name : running
  compose:
    ansible_host: public_ip_address
  ```
  
  The above code will fetch all the EC2 instances which are in running state in `us-est-2` region

**Step 2:- Creating configure file for Ansible**

* Create `ansible.cfg` file and add below code in it

  ```
  [defaults]
  enable_plugins=aws_ec2
  private_key_file = ~/.ssh/id_rsa
  ```
  
**Step 3:- Creating file for clustering**

* Create `cluster.yaml` file and add below code in it 

  ```
  ---
  - hosts: tag_Name_master
    user: ec2-user
    gather_facts: True
    become: yes
    become_method: sudo
    tasks:
      - name: master_ip
        set_fact:
          master: "{{ hostvars[inventory_hostname]['ansible_host'] | to_json }}"
      - name: master
        shell: /home/ec2-user/splunk/bin/splunk edit cluster-config -mode master -replication_factor 2 -search_factor 2 -secret 66546654 -auth admin:66546654 ; /home/ec2-user/splunk/bin/splunk restart ;
      - name: pause
        pause:
          minutes: 1

  - hosts: tag_Name_indexer*
    user: ec2-user
    become: yes
    become_method: sudo
    tasks:
      - name: master_ip
        set_fact:
          master: "{{ hostvars[groups['tag_Name_master'][0]]['ansible_host'] | to_json }}"
      - name: Indexer
        shell: /home/ec2-user/splunk/bin/splunk edit cluster-config -mode slave -master_uri https://"{{master}}":8089 -replication_port 4598 -secret 66546654 -auth admin:66546654 ; /home/ec2-user/splunk/bin/splunk restart ;
    gather_facts: True

  - hosts: tag_Name_search*
    user: ec2-user
    gather_facts: True
    become: yes
    become_method: sudo
    tasks:
      - name: master_ip
        set_fact:
          master: "{{ hostvars[groups['tag_Name_master'][0]]['ansible_host'] | to_json }}"
      - name: Search Head
        shell: /home/ec2-user/splunk/bin/splunk edit cluster-config -mode searchhead -master_uri https://"{{master}}":8089 -secret 66546654 -auth admin:66546654 ; /home/ec2-user/splunk/bin/splunk restart ;

  - hosts: tag_Name_forwarder*
    user: ec2-user
    gather_facts: True
    become: yes
    become_method: sudo
    tasks:
      - name: indexer_ip_1
        set_fact:
          indexer1: "{{ hostvars[groups['tag_Name_indexer1'][0]]['ansible_host'] | to_json }}"
      - name: indexer_ip_2
        set_fact:
          indexer2: "{{ hostvars[groups['tag_Name_indexer2'][0]]['ansible_host'] | to_json }}"
      - name: indexer1
        shell: /home/ec2-user/splunkforwarder/bin/splunk add forward-server {indexer1}:9997 ; /home/ec2-user/splunkforwarder/bin/splunk restart ;
      - name: indexer2
        shell: /home/ec2-user/splunkforwarder/bin/splunk add forward-server {indexer2}:9997 ; /home/ec2-user/splunkforwarder/bin/splunk restart ;
  ```
  
  Run below command to list all the details of instances
  ```
  ansible-inventory -i aws_ec2.yaml --list
  ```
  
  Run below command to create the indexer clustering
  ```
  ansible-playbook -i aws_ec2.yaml -u ec2-user cluster.yaml
  ```

  The above commands will create indexer clustering and will add all the indexers as the forwarding server for forwarders
  
  ![5](https://github.com/DhruvinSoni30/Terraform_Ansible_Splunk_Cluster/blob/main/images/2.png)

  That's it! Our setup is ready!
