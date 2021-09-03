# Deploying HeyEmoji using BitOps

## Context
Explain what we are wanting to accomplish and why we might want to accomplish it

<hr/>

## Required tools
1. [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git_)
1. [docker](https://docs.docker.com/get-docker/)
1. [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) - Install V1, some features we will be using aren't supported by the v2 api
1. [slack](https://slack.com/intl/en-ca/help/articles/209038037-Download-Slack-for-Windows)

This tutorial involves a small EKS cluster and deploying an application to it. Because of this, there will be AWS compute charges for completing this tutorial.

If you prefer skipping ahead to the final solution, the code created for this tutorial is on [Github](https://github.com/PhillypHenning/heyemoji).

<hr/>

## Instructions

### **Setting up our operations repo**
On your local machine create a folder called `operations-heyemoji`. Change directories into the newly created folder and using yeoman create a environment folder. Install yeoman and generator-bitops with the following;

```
npm install -g yo
npm install -g @bitovi/generator-bitops
```

Run yo @bitovi/bitops to create an operations repo. When prompted, name your environment “test”, answer “Y” to Terraform and Ansible, and “N” to the other supported tools.

```
yo @bitovi/bitops
```

![Alt Text](https://www.bitovi.com/hs-fs/hubfs/DevOps%20Page/Blogs/bitops-eks.gif?width=1348&name=bitops-eks.gif)

### **What is an operation repo?**

Before we continue let's quickly review what an ops repo is and where it belongs. Imagine this classic example; We're cooking a recipe from a cookbook. The recipe is made up of 2 components. 
1. The ingredients
1. The instructions

The ingredients as a whole, would be considered the application. Each ingredient is a specialized component which will be used to achieve a desired result. It's great having these ingredients, but without knowing what to do with them, they won't be very tasty (or useful) to us. 

This is why we need instructions (An OpsRepo)! The instructions explain how to cook, cut and combine our raw ingredients and transform them into a fully working recipe that we can enjoy with others.

So let's recap the "what is"; We have our application, which is made up of small specialized components, which work together to achieve a desired result. And we have our OpsRepo which explains how to combine the raw ingredients("build"), as well as serve("deploy") it so others so that they may try it themselves. Of course there is a little bit more nuance than that but this gets the point across. 

Now let's answer the "where". When creating an operation repo, much like a recipe, there are a few ways of packaging and presenting. Using the same analogy as above; We could package the instructions and ingredients together, a la a daily food delivery system (prepared meal kits for example), but by doing it this way the ingredients and instructions become intimately tied together, a change to one must be represented immediately by the other. An alternative approach is to package the ingredients and instructions separately. The benefit to this approach is that an update to the instructions doesn't effect the ingredients and vice versa. The instructions would tell you what ingredients you would need from the grocery store 

In this tutorial we will be following best practices by keeping our application and operation repos separate. 

### ***Configure Terraform*** 
### **Managing Terraform State files**
Either replace the contents of test/terraform/bitops.before-deploy.d/my-before-script.sh or create a new file called `create-tf-bucket.sh` with the content of;

#### `terraform/bitops.before-deploy.d/create-tf-bucket.sh`
```
aws s3api create-bucket --bucket $TF_STATE_BUCKET --region $AWS_DEFAULT_REGION --create-bucket-configuration LocationConstraint=$AWS_DEFAULT_REGION
```

Any shell scripts in test/terraform/bitops.before-deploy.d/ will execute before any Terraform commands. This script will create an S3 bucket with the name of whatever we set the TF_STATE_BUCKET environment variable to.

We will need to pass in TF_STATE_BUCKET when creating a BitOps container. S3 bucket names need to be globally unique.


### **Terraform Providers**
Either delete and create or rename `main.tf` to `create-tf-bucket.sh`.

Replace the `bucket = "YOUR BUCKET NAME" with the bucket name that was specified in the above step. 

Providers are integrations (usually created/maintained by the company that owns the integration) that instructs Terraform on how to perform requests. You can think of them like node modules, or python packages.

#### `terraform/providers.tf`
```
terraform {
    required_version = ">= 0.12"
    backend "s3" {
        bucket = "YOUR_BUCKET_NAME"
        key = "state"
    }
}

provider "local" {
    version = "~> 1.2"
}

provider "null" {
    version = "~> 2.1"
}

provider "template" {
    version = "~> 2.1"
}

provider "aws" {
    version = ">= 2.28.1"
    region  = "us-east-2"
}
```

### **Terraform variables**
Next create a `vars.tf` file.

We are going to put any variables that Terraform will be using here to consolidate the configuration settings.

#### `terraform/vars.tf`
```
/* set up env variables */
variable "AWS_DEFAULT_REGION" {
    type        = string
    description = "AWS region"
}
variable "TF_STATE_BUCKET" {
    type        = string
    description = "Terraform state bucket"
}
```


### **AWS Virtual Private Cloud**
Create another file called `vpc.tf`

Here we are going to configure our AWS Virtual Private Cloud which our EC2 instance will be hosted within.

#### `terraform/vpc.tf`
```
/* get region from AWS_DEFAULT_REGION */
data "aws_region" "current" {}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = merge(local.aws_tags,{
    Name = "heyemoji-blog-vpc"
  })
}

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id
  tags = local.aws_tags
}

resource "aws_subnet" "main" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = aws_vpc.main.cidr_block
  availability_zone = "${data.aws_region.current.name}a"
  tags = local.aws_tags
}

resource "aws_route_table" "rt" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }
  tags = local.aws_tags
}

resource "aws_route_table_association" "mfi_route_table_association" {
  subnet_id      = aws_subnet.main.id
  route_table_id = aws_route_table.rt.id
}
```

### **AWS ami**
Next we will create a Amazon Machine Image or ami, this provides information required to launch instances within AWS.


#### `terraform/ami.tf`
```
data "aws_ami" "ubuntu" {
  most_recent = true
  owners = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

### **AWS Security Groups**
So now we need to tell AWS what permissions our ec2 instance has. In the case below we are opening up SSH as well as websocket traffic. We don't need to make our instance accessible from the outer world, so this will work.  

#### `terraform/security-groups.tf`
```
/* local vars */
locals {
  aws_tags = {
    RepoName = " mmcdole/heyemoji"
    OpsRepoEnvironment = "blog-test"
    OpsRepoApp = "heyemoji-blog"
  }
}


resource "aws_security_group" "allow_traffic" {
  name        = "allow_traffic"
  description = "Allow traffic"
  vpc_id      = aws_vpc.main.id
  ingress = [{
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    ipv6_cidr_blocks = null
    prefix_list_ids = null
    security_groups = null
    self = null

  },{
    description = "WEBSOCKET"
    from_port   = 3334
    to_port     = 3334
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    ipv6_cidr_blocks = null
    prefix_list_ids = null
    security_groups = null
    self = null
  }]
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = merge(local.aws_tags,{
    Name = "heyemoji-blog-sg"
  })
}
```


### **AWS EC2 Instance**
And finally for the Terraform files, create the `instance.tf` file.
This tells Terraform that we are looking to provision a simple t3.micro ec2 instance that will contain our application.

#### `terraform/instance.tf`
```
resource "tls_private_key" "key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "aws_key" {
  key_name   = "heyemoji-blog-ssh-key"
  public_key = tls_private_key.key.public_key_openssh
}

resource "aws_instance" "server" {
  ami                         = data.aws_ami.ubuntu.id
  instance_type               = "t3.micro"
  key_name                    = aws_key_pair.aws_key.key_name
  associate_public_ip_address = true
  subnet_id                   = aws_subnet.main.id
  vpc_security_group_ids      = [aws_security_group.allow_traffic.id]
  monitoring                  = true
}
```

### ***Configure Ansible***
### **A quick note on images**
In this tutorial we are going to clone, build and deploy our image using Ansible.


### **Clean up generated files**
Some of the files that were generated aren't in scope for this blog. Please delete the following generated files/folders;
1. bitops.after.deploy
1. bitops.before.deploy
1. inventory.yml

### **Create Ansible Playbook**
Let's define our Ansible Playbook, for those unfamiliar this will be our automation blueprint, we will specify which tasks we want to run here. 

Create the following files within the `Ansible/` folder. 
#### `Ansible/playbook.yaml`
```
- hosts: bitops_servers
  become: true
  vars_files:
    - vars/default.yml
  tasks:
  - name: Include install
    include_tasks: tasks/install.yml
  - name: Include fetch
    include_tasks: tasks/fetch.yml
  - name: Include build
    include_tasks: tasks/build.yml
  - name: Include start
    include_tasks: tasks/start.yml
  - debug: 
      msg: "Hello from Ansible!"
```




### **Create Ansible Configuration**

### **Create Ansible Tasks**
### **Create Ansible Variables**




### **Create Slack Bot and add to Workspace**
(Instructions taken from the HeyEmoji README)
1. Browse to https://api.slack.com/apps?new_classic_app=1
2. Assign a name and workspace to your new Slack Bot Application
3. `Basic Information` > Set display name and icon
4. `App Home` > Add Legacy Bot User
5. `OAuth & Permissions` > Install App to Workspace
6. Copy your **Bot User OAuth Access Token** for your `HEY_SLACK_API_TOKEN`
7. Run `heyemoji` specifying the above token! 🎉

### Create a Github Workflow

### Update Github Secrets

### **Deploy HeyEmoji using BitOps + Github Workflows**
```
docker run \
-e ENVIRONMENT="test" \
-e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
-e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
-e AWS_DEFAULT_REGION="us-east-2" \
-e TF_STATE_BUCKET="heyemoji_blog" \
-e HEYMOJI_SLACK_API_TOKEN="YOUR SLACK API" \
-e TERRAFORM_DESTROY=true \
-v $(pwd):/opt/bitops_deployment \
bitovi/bitops:latest
```

### Verify deployment



```
docker run \
-e ENVIRONMENT="test" \
-e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
-e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
-e AWS_DEFAULT_REGION="us-east-2" \
-e TF_STATE_BUCKET="heyemoji-blog" \
-e HEYMOJI_SLACK_API_TOKEN="xoxb-2374139173-2444042423318-677piNpfTeDktszIjq46CfwX" \
-v $(pwd):/opt/bitops_deployment \
bitovi/bitops:latest
```

typically, we’d want to separately build the image and publish to a container registry rather than pull the code and build in place