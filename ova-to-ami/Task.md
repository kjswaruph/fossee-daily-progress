# Export VirtualBox OVA to AWS AMI

This guide walks you through exporting a VirtualBox VM as an OVA and importing it into AWS as an AMI, then launching EC2 instances. The steps follow a clear, numbered format similar to other docs in this repository.

## Prerequisites

- VirtualBox VM prepared and tested
- AWS account with access to S3 and EC2
- AWS CLI installed and configured (`aws configure`)
- S3 bucket in the target region for OVA storage
- Proper IAM role and policies for VM Import/Export

## 1. VirtualBox setup

- Rocky Linux 10
- With root user and online-test

### Update and install

```sh
useradd -m online-test
usermod -aG wheel online_test
passwd online-test # Set a temporary password, will be cleared later
# exit and login with online-test user

sudo dnf update -y

# Install docker
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker $USER

# Install git and cloud-init
sudo dnf install git cloud-init -y
```

### Install online-test

```sh
# Clone repo
git clone
cd online_test

# Uncomment everyline inside .env
cp .sameple.env .env
cp requirements/*.txt docker/Files/
```

Edit Dockerfile_codeserver and Dockerfile_django

```sh
# Replace FROM 18.04 with 20.04
FROM ubuntu:20.04

# Add this
ENV DEBIAN_FRONTEND=noninteractive
```

Edit Dockerfile_django

```sh
# Replace /tmp/requirements-py3.txt with
RUN cd /Sites/online_test && pip3 install -r /tmp/requirements-production.txt
```

Edit online_test/settings.py

```sh
# Edit line 31
ALLOWED_HOSTS = ["*"]
```

Create yaksh_data directory

```sh
mkdir -p yaksh_data/data
mkdir -p yaksh_data/output
cp -r yaksh yaksh_data/
cp requirements/requirements-codeserver.txt yaksh_data/
```

Add port

```sh
sudo firewall-cmd --permanent --add-port=8000/tcp
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

### Run compose

```sh
docker compose up -d
docker exec -i yaksh_django python3 manage.py makemigrations
docker exec -i yaksh_django python3 manage.py makemigrations notifications_plugin
docker exec -i yaksh_django python3 manage.py migrate
docker exec -i yaksh_django python3 manage.py loaddata demo_fixtures.json
docker exec -i yaksh_django python3 manage.py collectstatic --noinput
```

### Create service file

```sh
docker compose down
sudo vi /etc/systemd/system/yaksh.service
```

Paste the following

```
[Unit]
Description=yaksh service
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/online-test/online_test/docker/
User=online-test
Group=online-test
ExecStart=docker compose up -d
ExecStop=docker compose down

[Install]
WantedBy=multi-user.target
```

Start the service file

```sh
sudo systemctl enable --now yaksh
sudo systemctl status yaksh
```

### Cleanup vm

Edit `/etc/ssh/sshd_config` and set PermitRootLogin no

Remove password from online-test user

```sh
sudo passwd -d online_test
```

Edit sudoers

```sh
sudo visudo

# Add this line
%wheel ALL=(ALL) NOPASSWD: ALL
```

Edit `/etc/cloud/cloud.cfg`

```sh
system_info:
  # This will affect which distro class gets used
  distro: rhel
  # Default user name + that default users groups (if added/used)
  default_user:
    name: online-test
    ...other config
```

Delete connections

```sh
sudo rm -f /etc/ssh/ssh_host_*
sudo rm -f /etc/NetworkManager/system-connections/*
sudo rm -f /etc/udev/rules.d/70-persistent-net.rules
rm -rf ~/.ssh
```

Clean packages

```sh
sudo dnf update -y
sudo dnf autoremove -y
sudo dnf clean all
sudo rm -rf /var/cache/dnf
```

Clean logs and temp files

```sh
sudo journalctl --rotate
sudo journalctl --vacuum-time=1s
sudo rm -rf /var/log/* /tmp/* /var/tmp/*
```

```sh
sudo cloud-init clean --logs
sudo sync
history -c
cat /dev/null > ~/.bash_history
sudo poweroff
```

Export as OVA file

```sh
VBoxManage list vms #find your vm
VBoxManage export "your_vm" -o ~/rocky10-image.ova
```

## 2. AWS

- AWS Cli

### Create a S3 bucket

Create an S3 bucket in your target region to store the OVA file:

```sh
aws s3api create-bucket \
  --bucket your_bucket_name \
  --region your_region \
  --create-bucket-configuration LocationConstraint=your_region

# Optional: enable default encryption
aws s3api put-bucket-encryption \
  --bucket your_bucket_name \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

### Create IAM role

Create a IAM role named vmimport

> Note: You must enable STS in the region where you plan to store ova file

```sh
# Login to aws using aws cli
aws login # or aws configure
```

Create file `trust-policy.json` with following content

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "vmie.amazonaws.com" },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:Externalid": "vmimport"
        }
      }
    }
  ]
}
```

Create role by running

```sh
aws iam create-role --role-name vmimport --assume-role-policy-document "file://trust-policy.json"
```

Create file `role-policy.json` with following content

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetBucketLocation", "s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::your_bucket_name",
        "arn:aws:s3:::your_bucket_name/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:PutObject",
        "s3:GetBucketAcl"
      ],
      "Resource": [
        "arn:aws:s3:::your_bucket_name",
        "arn:aws:s3:::your_bucket_name/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:ModifySnapshotAttribute",
        "ec2:CopySnapshot",
        "ec2:RegisterImage",
        "ec2:Describe*"
      ],
      "Resource": "*"
    }
  ]
}
```

Attach policy to role

```sh
aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document "file://role-policy.json"
```

Upload ova to S3

```sh
aws s3 cp your_ova_file.ova s3://your_bucket_name/
aws s3 ls s3://your_bucket_name/
```

Import image

```sh
aws ec2 import-image \
    --description "$(date '+%b %d %H:%M') My server VM" \
    --disk-containers '[{
    "Format": "OVA",
    "UserBucket": {
      "S3Bucket": "your_bucker_name",
      "S3Key": "your_ova_file"
    }
  }]'
```

Monitor import task

```sh
aws ec2 describe-import-image-tasks --import-task-ids your_import_task_id
```

Wait till Status shows completed

Once the import task is completed, navigate to EC2>Images>AMIs and find your ami which is imported.

## Ansible

Install Ansible and create a Python virtual environment:

```sh
sudo dnf install -y python3-venv
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install ansible amazon.aws community.aws boto3 botocore
```

Get your AWS access and secret key

```sh
export AWS_ACCESS_KEY_ID=your_access_id
export AWS_SECRET_ACCESS_KEY=your_secret_key
export AWS_DEFAULT_REGION=your_aws_region
```

Create playbooks

`aws.yml`

```yaml
---
project_name: your_project_name
aws_region: your_region
instance_count: 5
instance_type: t3.small
ami_id: your_ami_id
keypair_name: your_keypairs

vpc_id: "your_default_vpc"
subnet_ids:
  - "your_default_subnet_a"
  - "your_default_subnet_b"
```

`keypair.create.yml`

```sh
- name: Import EC2 key pair
  hosts: localhost
  gather_facts: false
  vars:
    ansible_python_interpreter: "{{ playbook_dir }}/venv/bin/python"
  vars_files:
    - aws.yml

  tasks:
    - name: Import local SSH key into AWS
      amazon.aws.ec2_key:
        name: "{{ keypair_name }}"
        region: "{{ aws_region }}"
        key_material: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
```

`sg.create.yml`

```sh
---
- name: Create security groups
  hosts: localhost
  gather_facts: false
  vars:
    ansible_python_interpreter: "{{ playbook_dir }}/venv/bin/python"
  vars_files:
    - aws.yml
  tasks:
    - name: Create ALB security group
      amazon.aws.ec2_security_group:
        name: "{{ project_name }}-alb-sg"
        description: ALB SG
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            ports: 80
            cidr_ip: 0.0.0.0/0
      register: alb_sg

    - name: Create EC2 security group
      amazon.aws.ec2_security_group:
        name: "{{ project_name }}-ec2-sg"
        description: EC2 SG
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            ports: 8000
            group_id: "{{ alb_sg.group_id }}"
          - proto: tcp
            ports: 22
            cidr_ip: 0.0.0.0/0
      register: ec2_sg

    - name: Set security group ids
      ansible.builtin.set_fact:
        alb_sg_id: "{{ alb_sg.group_id }}"
        ec2_sg_id: "{{ ec2_sg.group_id }}"
    - name: Persist security group outputs
      copy:
        dest: ./out.sg.yml
        content: |
          alb_sg_id: {{ alb_sg.group_id }}
          ec2_sg_id: {{ ec2_sg.group_id }}
```

`sg.create.yml` generates out.sg.yml it containes security group id's of alb and ec2 instances

`ec2.create.yml`

```sh
---
- name: Create EC2 instances
  hosts: localhost
  gather_facts: false
  vars:
    ansible_python_interpreter: "{{ playbook_dir }}/../venv/bin/python"
  vars_files:
    - aws.yml
    - out.sg.yml

  tasks:
    - name: Launch EC2 instances
      amazon.aws.ec2_instance:
        name: "{{ project_name }}"
        key_name: "{{ keypair_name }}"
        image_id: "{{ ami_id }}"
        instance_type: "{{ instance_type }}"
        count: "{{ instance_count }}"
        region: "{{ aws_region }}"
        security_group: "{{ ec2_sg_id }}"
        vpc_subnet_id: "{{ subnet_ids[0] }}"

        volumes:
          - device_name: /dev/sda1
            ebs:
              delete_on_termination: true
        network:
          assign_public_ip: true
        tags:
          Project: "{{ project_name }}"
        termination_protection: false
        wait: true
      register: ec2
```

`alb.create.yml`

```sh
---
- name: Create ALB and Target Group
  hosts: localhost
  gather_facts: false
  vars:
    ansible_python_interpreter: "{{ playbook_dir }}/../venv/bin/python"
  vars_files:
    - aws.yml
    - out.sg.yml

  tasks:
    - name: Create target group
      community.aws.elb_target_group:
        name: "{{ project_name }}-tg"
        protocol: http
        port: 8000
        vpc_id: "{{ vpc_id }}"
        health_check_protocol: http
        health_check_path: /exam/login
        successful_response_codes: "200-399"
        stickiness_enabled: true
        stickiness_type: lb_cookie
        stickiness_lb_cookie_duration: 43200
        region: "{{ aws_region }}"
        state: present
        modify_targets: false
      register: tg

    - name: Create Application Load Balancer
      community.aws.elb_application_lb:
        name: "{{ project_name }}-alb"
        subnets: "{{ subnet_ids }}"
        security_groups:
          - "{{ alb_sg_id }}"
        scheme: internet-facing
        listeners:
          - Protocol: HTTP
            Port: 80
            DefaultActions:
              - Type: forward
                TargetGroupArn: "{{ tg.target_group_arn }}"
        region: "{{ aws_region }}"
        state: present
      register: alb

    - name: Fetch EC2 instances by tag
      amazon.aws.ec2_instance_info:
        region: "{{ aws_region }}"
        filters:
          "tag:Project": "{{ project_name }}"
          instance-state-name: running
      register: ec2_info

    - name: Register EC2 instances to Target Group
      community.aws.elb_target:
        target_group_arn: "{{ tg.target_group_arn }}"
        target_id: "{{ item.instance_id }}"
        target_port: 8000
        region: "{{ aws_region }}"
        state: present
      loop: "{{ ec2_info.instances }}"
```

`delete.yml`

```yaml
---
- name: Terminate everything
  hosts: localhost
  gather_facts: false
  vars:
    ansible_python_interpreter: "{{ playbook_dir }}/./venv/bin/python"
  vars_files:
    - aws.yml

  tasks:
    - name: Terminate EC2 Instances
      amazon.aws.ec2_instance:
        filters:
          "tag:Project": "{{ project_name }}"
        state: absent
        region: "{{ aws_region }}"
        wait: true
      register: ec2_term

    - name: Delete Load Balancer
      community.aws.elb_application_lb:
        name: "{{ project_name }}-alb"
        state: absent
        region: "{{ aws_region }}"
        wait: true

    - name: Delete Target Group
      community.aws.elb_target_group:
        name: "{{ project_name }}-tg"
        state: absent
        region: "{{ aws_region }}"

    - name: Get EC2 Security Group Info
      amazon.aws.ec2_security_group_info:
        region: "{{ aws_region }}"
        filters:
          group-name: "{{ project_name }}-ec2-sg"
      register: ec2_sg_info

    - name: Get ALB Security Group Info
      amazon.aws.ec2_security_group_info:
        region: "{{ aws_region }}"
        filters:
          group-name: "{{ project_name }}-alb-sg"
      register: alb_sg_info

    - name: Delete EC2 Security Group
      amazon.aws.ec2_security_group:
        group_id: "{{ item.group_id }}"
        state: absent
        region: "{{ aws_region }}"
      loop: "{{ ec2_sg_info.security_groups }}"
      when: ec2_sg_info.security_groups | length > 0
      retries: 5
      delay: 10
      register: ec2_sg_result
      until: ec2_sg_result is not failed

    - name: Delete ALB Security Group
      amazon.aws.ec2_security_group:
        group_id: "{{ item.group_id }}"
        state: absent
        region: "{{ aws_region }}"
      loop: "{{ alb_sg_info.security_groups }}"
      when: alb_sg_info.security_groups | length > 0
      retries: 10
      delay: 15
      register: alb_sg_result
      until: alb_sg_result is not failed
```

Run playbooks in order (from the playbooks directory):

```sh
# 1) Import your SSH public key as an EC2 key pair
ansible-playbook keypair.create.yml

# 2) Create security groups (ALB and EC2)
ansible-playbook sg.create.yml

# 3) Launch EC2 instances from your AMI
ansible-playbook ec2.create.yml

# 4) Create ALB and register targets
ansible-playbook alb.create.yml

# Cleanup (terminates instances, removes ALB/TG/SGs)
ansible-playbook delete.yml
```
