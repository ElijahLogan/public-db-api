
# Postgres and Express api hosted on EC2 container 
### Author

- [@elijahlogan](https://www.linkedin.com/in/elijah-logan/)


## Descriptioon

Create database and api on an EC2 server to create and store data cheaply on aws 


## Goal 

This project creates a ec2 instance inside a VPC that is accessible through the internet via configured subnet -> security group -> route table -> internet gateway

After Ec2 instance is created we will then create a database and make it's data accesible over the internet via api 


# Main file   
- Here we're creating a vpc with an ec2 instance inside of it  
- Creating a subnet to put the ec2 instance inside of to relieve future network congestion 
- Ceating and attaching a private key to our instance that we  will use to go inside the instance 
- Creating and attaching a security group to the instance that will allow data to come in through port 80 and ssh'ing over port 22 and data to leave the instance freely onto the internet

- Creating a router and internet gateway to facilitate the ec2's instance connection to the internet



```terraform

provider "aws" {
     region = "us-east-2"

 }
 

resource "tls_private_key" "demo_example" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "demo_generated_key" {
  key_name   = var.key_name
  public_key = tls_private_key.demo_example.public_key_openssh
 }
 
resource "aws_vpc" "app_vpc" {
  cidr_block = var.vpc_cidr
  instance_tenancy = "default"
  enable_dns_hostnames = true
  enable_dns_support = true

  tags = {
    Name = "app-vpc"
  }
}

resource "aws_internet_gateway" "demo_igw" {
  vpc_id = aws_vpc.app_vpc.id

  tags = {
    Name = "vpc_igw"
  }
}

resource "aws_route_table" "demo_public_rt" {
  vpc_id = aws_vpc.app_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.demo_igw.id
  }

  tags = {
    Name = "public_rt"
  }
}

resource "aws_subnet" "demo_public_subnet" {
  vpc_id            = aws_vpc.app_vpc.id
  cidr_block        = var.public_subnet_cidr
  map_public_ip_on_launch = true
  availability_zone = "us-east-2a"

  tags = {
    Name = "public-subnet"
  }
}




resource "aws_route_table_association" "demo_public_rt_asso" {
  subnet_id      = aws_subnet.demo_public_subnet.id
  route_table_id = aws_route_table.demo_public_rt.id
}

resource "aws_security_group" "demo_sg" {
  name        = "allow_ssh_http"
  description = "Allow ssh http inbound traffic"
  vpc_id      = aws_vpc.app_vpc.id

  ingress {
    description      = "SSH into isntance"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
  
    ingress {
    description      = "access express api"
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "allow_ssh_http"
  }
}

resource "aws_instance" "demo_web_db" {
  ami             = "ami-024e6efaf93d85776" 
  instance_type   = var.instance_type
  key_name        = aws_key_pair.demo_generated_key.key_name
  subnet_id       = aws_subnet.demo_public_subnet.id
  security_groups = [aws_security_group.demo_sg.id]


  tags = {
    Name = "web_db""
  }


}
```

# Variables 
## variables.tf

```terrafrom 
variable "instance_type" {
    type = string
    default  = "t3a.micro"
}


variable key_name {
        type = string
        default = "aws_ec2_pem_postgres"
}


variable "vpc_cidr" {
    type = string
    default = "178.0.0.0/22"
}
variable "public_subnet_cidr" {
    type = string
    default = "178.0.0.0/24"
}

```

 # outputs 
 ## outputs.tf 
 ```terraform
 output "web_instance_ip" {
    value = aws_instance.demo_web.public_ip
}

output "private_key" {
  value     = tls_private_key.demo_example.private_key_pem
  sensitive = true
}
```

## Deploy code 

- download code 
- cd into  it 
- add personal credentials to AWS provider 
- run terraform to initialize project, plan and ccreate resources  

```terraform 
terraform init
terraform plan -out=instance.plan
terraform apply instance.plan
```
### it should return two variables 

```bash 
private_key = <sensitive>
web_instance_ip = "<ip-address>
```

### print private key to copy for next steps

```bash 
terraform output private_key 
```

in your terminal create a folder with a file inside of it that is the same name as the private key name 

```bash 
mkdir example_folder
cd example_folder
vim aws_ec2_pem_postgres
```
**add private_key output to file then save with :wq**

-in example_folder run coomand to make file read only for secutiy reasons 

```bash
chmod 400 aws_ec2_pem_postgres 
```

## Connect to your instance using its Public DNS:

run in example_folder

```bash 
ssh -i "aws_ec2_pem_postgres.pem" ubuntu@ec2-<ip-address>.us-east-2.compute.amazonaws.com
ssh -i <PEM_FILE_NAME> OPERATING_SYSTEM@SERVER_IPV4_ADDRESS

```

# inside EC2: setting up postgres and express


## updating package manager then downloading postgres

Postgres packages are a part of Ubuntu’s default repositories so we can easily install it using apt packaging system.
We will also install one more package -contrib that will provide us some additional utility and functionality.


```bash

sudo apt-get update
sudo apt-get install postgresql postgresql-contrib
```

Now that we have successfully installed the software on our machine, let’s dive in and explore a bit.

The installation creates a user account postgres that is associated with the default Postgres role. We will log into this account to use Postgres.

Access the postgres account on your server by typing:


```bash 
sudo -i -u postgres
```


## Install and setup api resources 

### install node 

#### why node

```bash 
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs
```

### install pm2 


```bash
sudo npm install -g pm2
```


### initialize node package and download express


```bash
mkdir ex_api
cd ex_api 
npm init -y
npm i express 
```


