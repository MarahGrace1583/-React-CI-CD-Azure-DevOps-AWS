# React CI/CD Pipeline with Azure DevOps and AWS

This is my personal documentation of how I deployed a React application to an AWS Ubuntu VM using a fully automated Azure DevOps pipeline. I built this from scratch — provisioning the server, configuring Nginx, setting up the pipeline, and fixing a real memory crash along the way.

---

## What I Built

A 4-stage CI/CD pipeline that automatically builds, tests, publishes and deploys a React app to AWS every time I push to `main`.

```
Git push to main
       ↓
Azure DevOps Pipeline (self-hosted agent on my AWS VM)
       ↓
  Build → Test → Publish → Deploy
       ↓
React app live at http://13.222.28.116
```

---

## Tools I Used

| Tool | What I Used It For |
|------|-------------------|
| Terraform | Provisioned the Ubuntu EC2 instance on AWS |
| Ansible | Installed and configured Nginx on the VM |
| Azure DevOps | Created the CI/CD pipeline |
| Azure Repos | Hosted the React source code |
| Self-Hosted Agent | Ran the pipeline directly on my AWS VM |
| Nginx | Served the React build from /var/www/html |
| Node.js 18 | Built the React application |

---

## Step 1 — Provision the VM with Terraform

I created a project folder and wrote my Terraform config:

```bash
mkdir terraform
cd terraform
nano main.tf
```

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_security_group" "web_sg" {
  name        = "react-web-sg"
  description = "Allow SSH and HTTP"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "react_server" {
  ami                         = "ami-0c7217cdde317cfec"
  instance_type               = "t2.micro"
  subnet_id                   = "subnet-0de5a7836e219cc04"
  vpc_security_group_ids      = [aws_security_group.web_sg.id]
  associate_public_ip_address = true
  key_name                    = "devopskey"

  tags = {
    Name = "react-nginx-server"
  }
}

output "public_ip" {
  value = aws_instance.react_server.public_ip
}
```

> **Note:** I used a default subnet and VPC here for a quick setup. In a real production environment you would want a properly isolated VPC.

Then I ran:

```bash
terraform init
terraform apply
```

This gave me my VM's public IP: `13.222.28.116`

---

## Step 2 — Install Nginx with Ansible

```bash
mkdir ansible-nginx
cd ansible-nginx
sudo apt update
sudo apt install ansible -y
nano inventory.ini
```

```ini
[webservers]
13.222.28.116 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/devopskey.pem ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

```bash
nano nginx-setup.yml
```

```yaml
---
- name: Install and configure Nginx
  hosts: webservers
  become: yes

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Start and enable Nginx
      systemd:
        name: nginx
        state: started
        enabled: yes

    - name: Set permissions on /var/www/html
      file:
        path: /var/www/html
        owner: ubuntu
        group: ubuntu
        mode: '0755'
        recurse: yes
```

I fixed the key permission first:

```bash
chmod 400 ~/.ssh/devopskey.pem
```

First run failed because my laptop had never trusted the server before — SSH was blocking it for safety. I fixed this by SSHing in manually once first:

```bash
ssh -i ~/.ssh/devopskey.pem ubuntu@13.222.28.116
```

Then added `ansible_ssh_common_args='-o StrictHostKeyChecking=no'` to my inventory and ran again:

```bash
ansible-playbook -i inventory.ini nginx-setup.yml
```

---

## Step 3 — Import the React App into Azure Repos

```bash
git clone https://github.com/pravinmishraaws/my-react-app
cd my-react-app
git remote add azure <your-azure-repo-url>
git push azure main
```

---

## Step 4 — Set Up a Self-Hosted Agent on My VM

I created an agent pool called `SelfHostedPool` in Azure DevOps then SSH'd into my VM:

```bash
ssh -i ~/.ssh/devopskey.pem ubuntu@13.222.28.116
```

```bash
# Install Node.js
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Set up the agent
mkdir myagent && cd myagent
wget <agent-url-from-azure-devops>
tar zxvf vsts-agent-linux-x64-*.tar.gz
./config.sh

# Start as a background service
sudo ./svc.sh install
sudo ./svc.sh start
```

After this the agent showed as **Online** in Azure DevOps.

---

## Step 5 — Create SSH Service Connection

In Azure DevOps → Project Settings → Service connections → New → SSH:

- Host name: `13.222.28.116`
- Port: `22`
- Username: `ubuntu`
- Uploaded my `devopskey.pem` file
- Service connection name: `ubuntu-nginx-ssh`

---

## Step 6 — The Pipeline YAML

```yaml
trigger:
- main

stages:

# ================= BUILD =================
- stage: Build
  displayName: Build React App
  jobs:
  - job: BuildJob
    pool:
      name: Self-hosted
    steps:
    - checkout: self

    - task: UseNodeV1
      inputs:
        version: '18.x'
      displayName: Install Node.js

    - script: |
        npm install
        npm run build
      displayName: Build React App

    - publish: build
      artifact: react_build


# ================= TEST =================
- stage: Test
  displayName: Run Tests
  dependsOn: Build
  jobs:
  - job: TestJob
    pool:
      name: Self-hosted
    steps:
    - checkout: self

    - script: |
        npm install
        npm test -- --watchAll=false
      displayName: Run Tests


# ================= DEPLOY =================
- stage: Deploy
  displayName: Deploy to VM (Nginx)
  dependsOn: Test
  jobs:
  - job: DeployJob
    pool:
      name: Self-hosted
    steps:

    - download: current
      artifact: react_build

    - task: SSH@0
      inputs:
        sshEndpoint: 'ubuntu-nginx-ssh'
        runOptions: inline
        inline: 'sudo rm -rf /var/www/html/*'
      displayName: Clear Nginx Folder

    - task: CopyFilesOverSSH@0
      inputs:
        sshEndpoint: 'ubuntu-nginx-ssh'
        sourceFolder: '$(Pipeline.Workspace)/react_build'
        contents: '**'
        targetFolder: '/var/www/html'
      displayName: Copy Build to VM

    - task: SSH@0
      inputs:
        sshEndpoint: 'ubuntu-nginx-ssh'
        runOptions: inline
        inline: 'sudo systemctl restart nginx'
      displayName: Restart Nginx
```

---

## The Fix That Saved Everything — Swap Memory

My pipeline kept crashing during the build stage. The Node.js process was just dying with no clear error.

The problem: `t2.micro` only has ~1 GB RAM. React builds need more than that. When memory ran out, the OS killed the process.

My VM originally had:
- ~1 GB RAM
- 0 swap memory
- React build crashing at ~95% memory usage

I fixed it by adding 2 GB of swap:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

After this my system had:
- 1 GB RAM + 2 GB swap = ~3 GB usable memory
- Swap confirmed active with `free -h`
- Builds completed successfully every time after that

---

## Result

After all stages passed, I opened my browser and went to:

```
http://13.222.28.116
```

The React app was live. Every push to `main` now triggers the full pipeline automatically.

---

## What I Learned

- Terraform makes servers feel disposable in a good way — you can destroy and rebuild in minutes
- Ansible saves you from repeating manual server setup every time
- A self-hosted agent gives you more control than a Microsoft-hosted one when you need local access to deploy
- Always check memory on small VMs — swap space can be the difference between a working build and a crashed one
- CI/CD is about confidence as much as speed — every push goes through the same process, every time

---

*Built as part of my DevOps learning journey.*
