# ansible_aws_vpc
## This ansible playbook is used to deploy a complete VPC infrastructure of Three Instances one  Bastion, one webserver, and one for the Database server. Also it creates two Public subnets and one Private subnet (In different AZ), a NAT gateway, and allocates a new Elastic IP, route tables, and three Security groups within the VPC. It will also deploy three instances in different AZ along with different Security groups.
#### Prerequisite:

Install Ansible on an ec2 Instance and setup it as Ansible-master
Python boto library
Create an IAM Role with Policy AmazonEC2FullAccess and attach it to the Ansible master instance.
vim aws-vpc.yaml
```
---
- name: AWS VPC
  hosts: localhost

  vars:
    aws_region: us-east-1
    vpc_name: Ansible_vpc
    vpc_cidr_block: 172.16.0.0/16
    public_subnet_1_cidr: 172.16.0.0/20
    public_subnet_2_cidr:  172.16.16.0/20
    private_subnet_1_cidr: 172.16.32.0/20
    instance_type: t2.micro
    image: ami-0533f2ba8a1995cf9
    keypair: ansible-key
    bastion_exact_count: 1
    webserver_exact_count: 1
    dbserver_exact_count: 1

  tasks:

    - name: Creating VPC
      ec2_vpc_net:
        name:       "{{ vpc_name }}"
        cidr_block: "{{ vpc_cidr_block }}"
        region:     "{{ aws_region }}"
        state:      "present"
      register: my_vpc

    - name: Wait for 5 Seconds
      wait_for: timeout=5

    - name: Set VPC ID in variable
      set_fact:
        vpc_id: "{{ my_vpc.vpc.id }}"

    - name: Creating Public Subnet [AZ-1]
      ec2_vpc_subnet:
        state:  "present"
        vpc_id: "{{ vpc_id }}"
        cidr:   "{{ public_subnet_1_cidr }}"
        az:     "{{ aws_region }}a"
        region: "{{ aws_region }}"
        resource_tags:
          Name: "Public Subnet-1"
      register: my_public_subnet_az1

    - name: Set Public Subnet ID in variable [AZ-1]
      set_fact:
        public_subnet_az1_id: "{{ my_public_subnet_az1.subnet.id }}"

    - name: Creating Public Subnet [AZ-2]
      ec2_vpc_subnet:
        state:  "present"
        vpc_id: "{{ vpc_id }}"
        cidr:   "{{ public_subnet_2_cidr }}"
        az:     "{{ aws_region }}b"
        region: "{{ aws_region }}"
        resource_tags:
          Name: "Public Subnet-2"
      register: my_public_subnet_az2

    - name: Set Public Subnet ID in variable [AZ-2]
      set_fact:
        public_subnet_az2_id: "{{ my_public_subnet_az2.subnet.id }}"

    - name: Creating Private Subnet [AZ-1]
      ec2_vpc_subnet:
        state:  "present"
        vpc_id: "{{ vpc_id }}"
        cidr:   "{{ private_subnet_1_cidr }}"
        az:     "{{ aws_region }}c"
        region: "{{ aws_region }}"
        resource_tags:
          Name: "Private Subnet-1"
      register: my_private_subnet_az3

    - name: Set Private Subnet ID in variable [AZ-3]
      set_fact:
        private_subnet_az3_id: "{{ my_private_subnet_az3.subnet.id }}"

    - name: Create Internet Gateway for VPC
      ec2_vpc_igw:
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        state:  "present"
      register: my_vpc_igw

    - name: Set Internet Gateway ID in variable
      set_fact:
        igw_id: "{{ my_vpc_igw.gateway_id }}"

    - name: Creating a new NAT gateway and allocate new EIP 
      ec2_vpc_nat_gateway:
        state: present
        subnet_id: "{{ public_subnet_az2_id }}"
        wait: yes
        region:    "{{ aws_region }}"
        if_exist_do_not_create: true
      register: new_nat_gateway_az2

    - name: Set Nat Gateway ID in variable [AZ-2]
      set_fact:
        nat_gateway_az2_id: "{{ new_nat_gateway_az2.nat_gateway_id }}"

    - name: Wait for 5 Seconds
      wait_for: timeout=5

    - name: Set up public subnet route table
      ec2_vpc_route_table:
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        tags:
          Name: "Public"
        subnets:
          - "{{ public_subnet_az1_id }}"
          - "{{ public_subnet_az2_id }}"
        routes:
          - dest: "0.0.0.0/0"
            gateway_id: "{{ igw_id }}"

    - name: Set up private subnet route table [AZ-3]
      ec2_vpc_route_table:
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        tags:
          Name: "Private"
        subnets:
          - "{{ private_subnet_az3_id }}"
        routes:
          - dest: "0.0.0.0/0"
            gateway_id: "{{ nat_gateway_az2_id }}"

    - name: Creating Bastion Security Group
      ec2_group:
        name:  "Bastion-sg"
        description: "Bastion-sg"
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            from_port: 22
            to_port: 22
            cidr_ip: 0.0.0.0/0
        rules_egress:
          - proto: all
            cidr_ip: 0.0.0.0/0
      register: bastion_sg

    - name: Set Bastion Security Group ID in variable
      set_fact:
        bastion_sg_id: "{{ bastion_sg.group_id }}"

    - name: Creating Webserver Security Group
      ec2_group:
        name:  "Webserver-sg"
        description: "Webserver-sg"
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: 0.0.0.0/0
          - proto: tcp
            from_port: 443
            to_port: 443
            cidr_ip: 0.0.0.0/0
          - proto: "tcp"
            from_port: "22"
            to_port: "22"
            group_id: "{{ bastion_sg_id }}"
        rules_egress:
          - proto: all
            cidr_ip: 0.0.0.0/0

      register: webserver_sg

    - name: Set Main Security Group ID in variable
      set_fact:
        webserver_sg_id: "{{ webserver_sg.group_id }}"

    - name: Creating Database Server Security Group
      ec2_group:
        name: "DB-Server-sg"
        description: "DataBase Server-sg"
        vpc_id: "{{ vpc_id }}"
        region: "{{ aws_region }}"
        rules:
          - proto: "tcp"
            from_port: "22"
            to_port: "22"
            group_id: "{{ bastion_sg_id }}"
          - proto: "tcp"
            from_port: 3306
            to_port: 3306
            group_id: "{{ webserver_sg_id }}"
        rules_egress:
          - proto: all
            cidr_ip: 0.0.0.0/0

    - name: "Launching Webserver Instance"
      ec2:
        instance_type: "{{ instance_type }}"
        key_name: "{{ keypair }}"
        image: "{{ image }}"
        region: "{{ aws_region }}"
        group: Webserver-sg
        vpc_subnet_id: "{{ public_subnet_az1_id }}"
        assign_public_ip: yes
        wait: yes
        count_tag:
          Name: Webserver
        instance_tags:
          Name: Webserver
        exact_count: "{{ webserver_exact_count }}"

    - name: "Launching Bastion Instance"
      ec2:
        instance_type: "{{ instance_type }}"
        key_name: "{{ keypair }}"
        image: "{{ image }}"
        region: "{{ aws_region }}"
        group: Bastion-sg
        vpc_subnet_id: "{{ public_subnet_az2_id }}"
        assign_public_ip: yes
        wait: yes
        count_tag:
          Name: Bastion
        instance_tags:
          Name: Bastion
        exact_count: "{{ bastion_exact_count }}"

    - name: "Launching Database Instance"
      ec2:
        instance_type: "{{ instance_type }}"
        key_name: "{{ keypair }}"
        image: "{{ image }}"
        region: "{{ aws_region }}"
        group: DB-Server-sg
        vpc_subnet_id: "{{ private_subnet_az3_id }}"
        wait: yes
        count_tag:
          Name: DB-Server
        instance_tags:
          Name: DB-Server
        exact_count: "{{ dbserver_exact_count }}"
```
##### Executing the playbook - # ansible-playbook aws-vpc.yaml
