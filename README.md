# Deploying HeyEmoji using BitOps

## Context
In this tutorial we are going to deploy the HeyEmoji Slack app to AWS using Terraform and Ansible and orchestrating the tools using BitOps! 

HeyEmoji is a fantastic reward system teams can use to recognize each others accomplishments, dedication and hard work. BitOps is an Operations Repo orchestration tool which enables teams to write their infrastructure as code and deploy that code easily across multiple environments. 

<hr/>

## Required tools
1. [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git_)
1. [docker](https://docs.docker.com/get-docker/)
1. [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) - Install V1, some features we will be using aren't supported by the v2 api
1. [slack](https://slack.com/intl/en-ca/help/articles/209038037-Download-Slack-for-Windows)

This tutorial involves a small EKS cluster and deploying an application to it. Because of this, there will be AWS compute charges for completing this tutorial.

If you prefer skipping ahead to the final solution, the code created for this tutorial is on [Github](https://github.com/PhillypHenning/operations-heyemoji.git).

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

<hr />

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

Providers are integrations (usually created/maintained by the company that owns the integration) that instructs Terraform on how to perform a given request.

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
    RepoName = " <USERNAME>/heyemoji"
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
Create the `instance.tf` file.
This tells Terraform that we are looking to provision a simple t3.micro ec2 instance, sets the security groups (that we created above) and adds itself to the VPC network.

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

### Create Ansible inventory
And finally for the Terraform files, we are going to create an inventory and context file that Ansible will use. The `inventory.tmpl` will be loaded by the Ansible config, and the `locals.tf` file will inject the ip and ssh_keyfile values into the tmpl file during the Terraform apply stage. 

#### `terraform/inventory.tmpl`
```
heyemoji_blog_servers:
 hosts:
   ${ip} 
 vars:
   ansible_ssh_user: ubuntu
   ansible_ssh_private_key_file: ${ssh_keyfile}
```

#### `terraform/locals.tf`
```
resource "local_file" "private_key" {
  # This creates a keyfile pair that allows ansible to connect to the ec2 container
  sensitive_content = tls_private_key.key.private_key_pem
  filename          = format("%s/%s/%s", abspath(path.root), ".ssh", "heyemoji-blog-ssh-key.pem")
  file_permission   = "0600"
}

resource "local_file" "ansible_inventory" {
  content = templatefile("inventory.tmpl", {
    ip          = aws_instance.server.public_ip,
    ssh_keyfile = local_file.private_key.filename
  })
  filename = format("%s/%s", abspath(path.root), "inventory.yaml")
}
```



<hr />

### ***Configure Ansible***
### **A quick note on images**
In this tutorial we are going to clone, build and deploy our image using Ansible. It's worth noting that this doesn't have to be the same pattern you use, in fact, it is recommended that the application build is done separate from the operations repo. 

You may be wondering why we would want to separate the build and deployment steps. Well let's consider the OpsRepo once again, the reason we separate the Application and OpsRepos is to have distinct configurations and git histories. 

If the application repo handles the building of images, then a change to the application code would initiate the image build. The application repo and the application images have a logical flow to them. A change in the code, *could* kickoff a new build and create a new image that represents the change. 

Conversely, if the OpsRepo was to handle the application building, it would be responsible for initiating the new image build, but it wouldn't be aware of changes (not by default). Images built using the OpsRepo would pull the latest state of the code and build an image based on that. Though this isn't a wrong way to go about things, it adds unnecessary connections between the OpsRepo and AppRepo and sets up a vector of failure that wouldn't be there is the AppRepo handled its own image building. 

All this said, for ease of blog writing, we are going to keep our setup simple and manually pull and build the image. While we do that, keep the above in mind and see if you can find places where this setup may cause headaches. 

### **Clean up generated files**
Some of the files that were generated aren't in scope for this blog. Please delete the following generated files/folders;
1. bitops.after.deploy
1. bitops.before.deploy
1. inventory.yml

### **Create Ansible Playbook**
Let's define our Ansible Playbook. For those unfamiliar this will be our automation blueprint, we will specify which tasks we want to run here and define our tasks in their own files in a later section. 

Create the following files within the `Ansible/` folder. 
#### `ansible/playbook.yaml`
```
- hosts: heyemoji_blog_servers
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
Next, we are going to create an Ansible configuration file, this informs Ansible where the terraform generated inventory file can be found, as well as a few other notable setting flags. 


#### `ansible/inventory.cfg`
```
[defaults]
inventory=../terraform/inventory.yaml
host_key_checking = False
transport = ssh

[ssh_connection]
ssh_args = -o ForwardAgent=yes
```

### **Create Ansible Variables**
Next we are going to set up ENV vars we will use later in our Ansible tasks (next step). Make sure to update the USERNAME and REPO to represent your forked HeyEmoji path. 

#### `ansible/vars/default.yml`
```
heyemoji_repo: "https://github.com/<USERNAME>/heyemoji.git"
heyemoji_path: /home/ubuntu/heyemoji

heyemoji_bot_name:	heyemoji-dev
heyemoji_database_path:	./data/
heyemoji_slack_api_token: "{{ lookup('env', 'HEYEMOJI_SLACK_API_TOKEN') }}"
heyemoji_slack_emoji:	star:1
heyemoji_slack_daily_cap: "5"
heyemoji_websocket_port: "3334"

create_containers: 1
default_container_image: heyemoji:latest
default_container_name: heyemoji
default_container_image: ubuntu
default_container_command: /heyemoji
```


### **Create Ansible Tasks**
Alright now the fun part! We have to define our Ansible tasks, which are the specific instructions we want our playbook to execute on. In this tutorial we will need; a build, fetch, install and deploy task. 


#### `ansible/tasks/build.yml`

```
- name: build container image
  docker_image:
    name: "{{ default_container_image }}"
    build:
      path: "{{ heyemoji_path }}"
    source: build
    state: present
```

#### `ansible/tasks/fetch.yml`
```
- name: git clone heyemoji
  git:
    repo: "{{ heyemoji_repo }}"
    dest: "{{ heyemoji_path }}"
  become: no
```

#### `ansible/tasks/install.yml`
```
# install docker
- name: Install aptitude using apt
  apt: name=aptitude state=latest update_cache=yes force_apt_get=yes

- name: Install required system packages
  apt: name={{ item }} state=latest update_cache=yes
  loop: [ 'apt-transport-https', 'ca-certificates', 'curl', 'software-properties-common', 'python3-pip', 'virtualenv', 'python3-setuptools']

- name: Add Docker GPG apt Key
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: Add Docker Repository
  apt_repository:
    repo: deb https://download.docker.com/linux/ubuntu bionic stable
    state: present

- name: Update apt and install docker-ce
  apt: update_cache=yes name=docker-ce state=latest

- name: Install Docker Module for Python
  pip:
    name: docker
```

#### `ansible/tasks/start.yml`
```
# Creates the number of containers defined by the variable create_containers, using values from vars file
- name: Create default containers
  docker_container:
    name: "{{ default_container_name }}{{ item }}"
    image: "{{ default_container_image }}"
    command: "{{ default_container_command }}"
    exposed_ports: "{{ heyemoji_websocket_port }}"
    env:
      HEY_BOT_NAME:	"{{ heyemoji_bot_name }}"
      HEY_DATABASE_PATH: "{{ heyemoji_database_path }}"
      HEY_SLACK_TOKEN: "{{ heyemoji_slack_api_token }}"
      HEY_SLACK_EMOJI:	"{{ heyemoji_slack_emoji }}"
      HEY_SLACK_DAILY_CAP:	"{{ heyemoji_slack_daily_cap }}"
      HEY_WEBSOCKET_PORT:	"{{ heyemoji_websocket_port }}"
    # restart a container
    # state: started
  register: command_start_result
  loop: "{{ range(0, create_containers, 1)|list }}"
```

### **Create Slack Bot and add to Workspace**
Using the Instructions taken from the HeyEmoji README;
1. Browse to https://api.slack.com/apps?new_classic_app=1
2. Assign a name and workspace to your new Slack Bot Application
3. `Basic Information` > Set display name and icon
4. `App Home` > Add Legacy Bot User
5. `OAuth & Permissions` > Install App to Workspace
6. Copy your **Bot User OAuth Access Token** for your `HEYEMOJI_SLACK_API_TOKEN`
7. Run `heyemoji` specifying the above token! 🎉


### **Deploy HeyEmoji using BitOps + Github Workflows**
And with all the above in place we are finally ready to deploy our HeyEmoji Slack app!! Make sure to replace the "VALUES" to represent your credentials and tokens. 

```
docker run \
-e ENVIRONMENT="test" \
-e AWS_ACCESS_KEY_ID="AWS_ACCESS_KEY_ID" \
-e AWS_SECRET_ACCESS_KEY="AWS_SECRET_ACCESS_KEY" \
-e AWS_DEFAULT_REGION="us-east-2" \
-e TF_STATE_BUCKET="heyemoji_blog" \
-e HEYEMOJI_SLACK_API_TOKEN="YOUR SLACK API TOKEN" \
-v $(pwd):/opt/bitops_deployment \
bitovi/bitops:latest
```

### Verify deployment
Open Slack and create a new private channel. Add your new bot to the channel which you can do by @mentioning them in the channel's chat. Once the bot has been added type in chat `@HeyEmoji - Blog leaderboards`, a response should pop up saying; `Nobody has given any emoji points yet!`

This tells us our bot is alive!! You can now hand out awards to others in the chat by typing `Hey @member have a :star:`

### Cleanup
To delete the resources you've provisioned simply add `"-e TERRAFORM_DESTROY=true \"` to the docker run command

```
docker run \
-e ENVIRONMENT="test" \
-e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
-e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
-e AWS_DEFAULT_REGION="us-east-2" \
-e TF_STATE_BUCKET="heyemoji_blog" \
-e HEYEMOJI_SLACK_API_TOKEN="YOUR SLACK API TOKEN" \
-e TERRAFORM_DESTROY=true \
-v $(pwd):/opt/bitops_deployment \
bitovi/bitops:latest
```


### Final word
Great work! You've deployed the HeyEmoji slack app to your aws infrastructure using Terraform and Ansible, and we orchestrated the build + deploy using BitOps. We learned a few concepts such as what an OpsRepo is and what best practices there are when building our application images. 
