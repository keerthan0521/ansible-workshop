# ansible-workshop
Ansible Lab Steps
=================
Login to AWS Console


### Lab 1: Installation and Configuration of Ansible


- Launch instance RHEL 9 machine in us-east-1. Choose t2.micro. In security group, 
- allow SSH (22) and HTTP (80) for all incoming traffic. Add Tag Name: Ansible-ControlNode

- Once the EC2 is up & running, SSH into one of it and set the hostname as 'Control-Node'. 
`sudo hostnamectl set-hostname Control-Node`
- Now you can exit and login again. It will show the new hostname.
- or you can type 'bash' and open another shell which shows new hostname.

### Update the package repository with latest available versions
`sudo yum check-update`

### Install latest version of Python. 
`sudo yum install python3-pip wget
python3 --version
sudo pip3 install --upgrade pip`


### Install awscli, boto, boto3 and ansible
### Boto/Boto3 are AWS SDK which will be needed while accessing AWS APIs
`sudo pip3 install awscli boto boto3
sudo pip3 install ansible==4.10.0`

`pip show ansible`

`aws configure`
add AWS access key 
&
AWS secret Access key

******Create the Playbook for creating managed nodes*********
` vi ec2-playbook.yml`

`---
- hosts: localhost
  connection: local

  tasks:
    - name: Execute curl command to get token
      shell: "curl -X PUT 'http://169.254.169.254/latest/api/token' -H 'X-aws-ec2-metadata-token-ttl-seconds: 21600'"
      register: TOKEN

    - name: Get region of instance
      shell: "curl -H 'X-aws-ec2-metadata-token:{{ TOKEN.stdout }}' http://169.254.169.254/latest/meta-data/placement/region/"
      register: region

    - name: Get AMI ID of instance
      shell: "curl -H 'X-aws-ec2-metadata-token:{{ TOKEN.stdout }}' http://169.254.169.254/latest/meta-data/ami-id"
      register: ami_id

    - name: Get keypair of instance
      shell: "curl -H 'X-aws-ec2-metadata-token:{{ TOKEN.stdout }}' http://169.254.169.254/latest/meta-data/public-keys/| cut -c 3-100 "
      register: kp

    - name: Get Instance Type of instance
      shell: "curl -H 'X-aws-ec2-metadata-token:{{ TOKEN.stdout }}' http://169.254.169.254/latest/meta-data/instance-type"
      register: instance_type


    - name: Get subnet id of instance
      shell: "curl -H 'X-aws-ec2-metadata-token:{{ TOKEN.stdout }}' -v http://169.254.169.254/latest/meta-data/network/interfaces/macs/$(curl -H 'X-aws-ec2-metadata-token:{{ TOKEN.stdout }}' -v http://169.254.169.254/latest/meta-data/network/interfaces/macs)/subnet-id"
      register: subnet

    - name: Get security group of instance
      shell: "curl -H 'X-aws-ec2-metadata-token:{{ TOKEN.stdout }}' -v http://169.254.169.254/latest/meta-data/network/interfaces/macs/$(curl -H 'X-aws-ec2-metadata-token:{{ TOKEN.stdout }}' -v http://169.254.169.254/latest/meta-data/network/interfaces/macs)/security-group-ids/"
      register: secgrp


    - name: Generate SSH keypair
      openssh_keypair:
        force: yes
        path: /home/ec2-user/.ssh/id_rsa

    - name: Get the public key
      shell: cat /home/ec2-user/.ssh/id_rsa.pub
      register: pubkey

    - name: Create EC2 instance
      ec2:
        key_name: "{{ kp.stdout }}"
        group_id: "{{ secgrp.stdout }}"
        instance_type: "{{ instance_type.stdout }}"
        image: "{{ ami_id.stdout }}"         # "ami-0931978297f275f71"
        wait: true
        region: "{{ region.stdout }}"
        instance_tags:
          Name: "{{ item }}"
        vpc_subnet_id: "{{ subnet.stdout }}"
        assign_public_ip: yes
        user_data: |
           #!/bin/bash
           echo "{{ pubkey.stdout }}" >> /home/ec2-user/.ssh/authorized_keys
      register: ec2var
      loop:
          - managed-node-1
          - managed-node-2

    - name: Make ansible directory
      file:
        path: /etc/ansible
        state: directory
      become: yes

    - debug:
        msg: "{{ ec2var.results[0].instances[0].private_ip }}"

    - debug:
        msg: "{{ ec2var.results[1].instances[0].private_ip }}"`


save the file

ansible-playbook ec2-playbook.yml



### Once you get the ip addresses, do the following:
sudo vi /etc/ansible/hosts

### Add the prive IP addresses, by pressing "INSERT" 
node1 ansible_ssh_host=node1-private-ip ansible_ssh_user=ec2-user
node2 ansible_ssh_host=node2-private-ip ansible_ssh_user=ec2-user

e.g. node1 ansible_ssh_host=172.31.14.113 ansible_ssh_user=ec2-user
     node2 ansible_ssh_host=172.31.2.229 ansible_ssh_user=ec2-user


### Save the file using "ESCAPE + :wq!"

### list all managed node ip addresses.
ansible all --list-hosts

###  SSH into each of them and set the hostnames.
`ssh ec2-user@< Replace Node 1 IP >
sudo hostnamectl set-hostname managed-node-1
exit`

`ssh ec2-user@< Replace Node 2 IP >
sudo hostnamectl set-hostname managed-node-2
exit`

###  Use ping module to check if the managed nodes are able to interpret the ansible modules
`ansible all -m ping`

################################
Lab 2: Exploring Ad-Hoc Commands
################################

`sudo  vi /etc/ansible/hosts`
### Add the given line, by pressing "INSERT" 
#  # add localhost and add the connection as local so that it wont try to use ssh
`localhost ansible_connection=local`
### save the file using "ESCAPE + :wq!"

### In real life situations, one of the managed node may be used as the ansible control node.
###  In such cases, we can make it a managed node, by adding localhost in hosts inventory file.


### get memory details of the hosts using the below ad-hoc command
`ansible all -m command -a "free -h"`
OR
`ansible all -a "free -h"`

### Create a user ansible-new in the 2 nodes + the control node
### This creates the new user and the home directory /home/ansible-new
`ansible all -m user -a "name=ansible-new" --become`

### lists all users in the machine. Check if ansible-new is present in the managed nodes / localhost
`ansible node1 -a "cat /etc/passwd"`

### List all directories in /home. Ensure that directory 'ansible-new' is present in /home. 
`ansible node2 -a "ls /home"`


###  Change the permission mode from '700' to '755' for the new home directory created for ansible-new
`ansible node1 -m file -a "dest=/home/ansible-new mode=755" --become`


### Check if the permissions got changed
`ansible node1 -a "sudo ls -l /home"`


### Create a new file in the new dir in node 1
`ansible node1 -m file -a "dest=/home/ansible-new/demo.txt mode=600 state=touch" --become`


### Check if the permissions got changed
`ansible node1 -a "sudo ls -l /home/ansible-new/"
`

### Add content into the file
ansible node1 -b -m lineinfile -a 'dest=/home/ansible-new/demo.txt line="This server is managed by Ansible"'


### check if the lines are added in demo.txt
ansible node1 -a "sudo cat /home/ansible-new/demo.txt"


### You can remove the line using parameter state=absent
ansible node1 -b -m lineinfile -a 'dest=/home/ansible-new/demo.txt line="This server is managed by Ansible" state=absent'

### check if the lines are removed from demo.txt
ansible node1 -b -a "sudo cat /home/ansible-new/demo.txt"

# Now copy a file from ansible-control node to host node 1
touch test.txt
echo "This file will be copied to managed node using copy module" >> test.txt

ansible node1 -m copy -a "src=test.txt dest=/home/ansible-new/test" -b
### --become can be replaced by -b

### check if the file got copied to managed node.
ansible node1 -b -a "sudo ls -l /home/ansible-new/test"

sudo vi /etc/ansible/hosts

### Remove the below line from hosts inventory file. 
localhost ansible_connection=local

### save the file using "ESCAPE + :wq!"

================================================================================
Playbook
-----------------
vi first.yml

---
- name: first play
  hosts: all
  become: yes
  tasks:
    - name: create a directory
      file:
        path: /test
        state: directory
    - name: create a new file
      file:
        path: demo.txt
        mode: 0664
        state: touch

 save it

ansible-playbook first.yml

ansible all -m command -a "ls -l"

===========================================================
--------------
Lab3:
--------------
vi install-apache.yml
---
- name: This play will install apache web servers on all the hosts
  hosts: all
  become: yes
  tasks:
    - name: Task1 will install httpd using yum
      yum:
        name: httpd
        update_cache: yes
        state: latest
    - name: Task2 will upload custom index.html into all hosts
      copy:
       src: "index.html"
       dest: "/var/www/html/index.html"
    - name: Task3 will setup attributes for file
      file:
        path: /var/www/html/index.html
        owner: apache
        group: apache
        mode:  0644
    - name: Task4 will start the httpd
      service:
        name: httpd
        state: started

save the file

vi index.html
<html>
  <body>
  <h1>Welcome to CloudThat</h1>
  <img src= "https://d3ffutjd2e35ce.cloudfront.net/assets/logo1.png" >
  </body>
</html>

save the file

# ansible-playbook install-apache.yml

 # curl 10.142.0.24     (private ip of vm)
 # curl 10.142.0.23
=================================================================
--------------
Lab4:
------------------
vi putfile.yml
---
- hosts: all
  become: yes
  tasks:
    - name: Creating a new user cloudthat
      user:
        name: cloudthat
    - name: Creating a directory for the new user
      file:
        path: /home/cloudthat/test
        state: directory

save the file

### ansible-playbook putfile.yml


### ansible all -m command -a "tail -n 2 /etc/passwd"
###  ansible all -m command -a "ls -l /home/cloudthat" -b    (-b is become root user)


Lab4a:

vi p2.yml
---
- hosts: all
  become: yes
  tasks:
    -   name: Creating a new user test
        user:
          name: test
    -   name: Creating a directory for the new user
        file:
          path: /home/test/demo
          state: directory
    -   name: creating a folder named ansible
        file:
          path: /home/test/ansible
          state: directory
    -   name: creating a file within the folder ansible
        file:
          path: /home/test/ansible/hello.txt
          state: touch
    -   name: Changing owner and group with permission for the file within the folder named ansible
        file:
          path: /home/test/ansible/hello.txt
          owner: root
          group: test
          mode: 0665
    -   name: adding a block of string to the file created named hello.txt
        blockinfile:
          path: /home/test/ansible/hello.txt
          block: |
            This is line 1
            This is line 2

save the file

### ansible-playbook p2.yml

###  ansible all -a "sudo cat /home/test/ansible/hello.txt"


Task 2: Uninstalling Apache Service
=========================================

vi service.yml
---
- hosts: all
  become: yes
  tasks:
   - name: uninstalling httpd
     yum:
        name: httpd
        state: absent
   - name: Downloading a file
     get_url:
        url: https://s3.ap-south-1.amazonaws.com/files.cloudthat.training/devops/ansible-essentials/sql_permissions.txt
        dest: /tmp/
   - name: Disable SElinux without using command and shell
     selinux:
        state: disabled

save the file

# ansible-playbook service.yml
# ansible all -m command -a "yum list httpd" -b
# ansible all -m command -a "ls -l /tmp" -b
# ansible all -m command -a "getenforce" -b

=====================================================

Lab 5: Implementing Ansible Variables

Task 1: Configuring packages in ansible using variables
--------------------------------

vi implement-vars.yml
---
- hosts: '{{ hostname }}'
  become: yes
  vars:
    hostname: all
    package1: httpd
    destination: /var/www/html/index.html
    source: /home/ec2-user/lab5/file/index.html  #control node path
  tasks:
    - name: Install defined package
      yum:
        name: '{{ package1 }}'
        update_cache: yes   # refresh the caches before applying whatever change is necessary
        state: latest
    - name: Start desired service
      service:
        name: '{{ package1 }}'
        state: started
    - name: copy required index.html to the document folder for httpd.
      copy:
        src: '{{ source }}'
        dest: '{{ destination }}'

save the file


vi index.html

<html>
  <body>
  <h1>Welcome to CloudThat</h1>
  <h1>Welcome to variables</h1>
  </body>
</html>

save the file

ansible-playbook implement-vars.yml


view page use the public ip address of the vm 


Task 2: Implementing ansible variables using extra-vars option
----------------------------------------
### create new file in the same location

vi index1.html
<html>
  <body>
  <h1>This is the alternate Home Page</h1>
  <img src= "https://d3ffutjd2e35ce.cloudfront.net/assets/logo1.png" >
  </body>
</html>

[ec2-user@ansible file]$ pwd
/home/ec2-user/lab5/file
[ec2-user@ansible file]$


### ansible-playbook implement-vars.yml --extra-vars "source=/home/ec2-user/lab5/file/index1.html"


Task 3: Configuring variables as a separate file and implementing ansible playbook
--------------------------------

vi implement-vars1.yml
---
- hosts: '{{ hostname }}'
  become: yes
  vars_files:
    - myvariables.yml
  tasks:
    - name: Install defined package
      yum:
        name: '{{ package1 }}'
        update_cache: yes
        state: latest
    - name: Start desired service
      service:
        name: '{{ package1 }}'
        state: started
    - name: copy required index.html to the document folder for httpd.
      copy:
        src: '{{ source }}'
        dest: '{{ destination }}'

save the file

vi myvariables.yml
---
hostname: all
package1: httpd
destination: /var/www/html/index.html
source: /home/ec2-user/lab5/file/index.html


save the file

vi index.html
<html>
  <body>
  <h1>Welcome to CloudThat</h1>
  <h1>Welcome to variables</h1>
  </body>
</html>

### ansible-playbook implement-vars1.yml
=========================================================
Lab 6 Task Inclusion
==============================
add the line in the inventory

vi /etc/ansible/hosts

localhost ansible_connection=local

save the file


vi first.yaml
---
- hosts: localhost
  gather_facts: no
  become: yes
  tasks:
  - name: install common packages
    yum:
      name: [wget, curl]
      state: present

  - name: inclue task for httpd installation
    include_tasks: second.yaml

save the file

vi second.yaml
---
  - name: install the httpd package
    yum:
      name: httpd
      state: latest
      update_cache: yes

  - name: start the httpd service
    service:
      name: httpd
      state: started
      enabled: yes

save the file

now  run the first file

### ansible-playbook first.yaml


vi third.yaml
---
- hosts: localhost
  gather_facts: no
  become: yes
  tasks:
  - name: install common packages
    yum:
      name: [wget, curl]
      state: present
    register: out

  - name: list result of previous task
    debug:
      msg: "{{ out.rc}}"

  - name: inclue task for httpd installation
    include_tasks: second.yaml
    when: out.rc == 0


Now run the third file

ansible-playbook third.yaml
