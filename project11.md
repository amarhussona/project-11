# ANSIBLE CONFIGURATION MANAGEMENT

## Install and configure Ansible client to act as a Jump Server/Bastion Host

Install and configure Ansible client to act as a Jump Server/Bastion Host. To do this, `Jenkins` EC2 instance is updated to `Jenkins-Ansible`

![](./images/01_instance.png)

In GitHub create a new repository and name it `ansible-config-mgt`

Install Ansible on jenkins-ansible:

`sudo apt update`

`sudo apt install ansible`

![](./images/02_install_ansible.png)

Confirm Ansible version:

`ansible --version`

![](./images/03_ansible_version.png)

Create a new Freestyle project ansible in Jenkins and point it to ‘ansible-config-mgt’ repository:

![](./images/05_config_freestyle_project.png)

Configure Webhook in GitHub and set webhook to trigger ansible build:

![](./images/05_config_freestyle_project2.png)

Configure build trigger to GitHub hook trigger for GITScm polling and a Post-build job to save all files (**):

![](./images/06_config_freestyle_project3.png)

Make sure to use `/main` branch not master:

![](./images/07_use_main.png)

Test setup by making some change in README.MD file in main branch and make sure that builds starts automatically and Jenkins saves the files:

`ls /var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/`

![](./images/08_testing_jenkins_setup.png)
![](./images/09_verify_jenkins_works.png)

Note: Allocate Elastic IP to Jenkins- Ansible Server and update github webhook with elastic IP

## Prepare your development environment using Visual Studio Code

Configured VSC to connect to the newly created github repository and ssh into the EC2 instances

![](./images/17_config_vscode.png)

Ansible uses TCP port 22 by default,which means it needs to ssh into target servers. Add privatekey.pem file to ssh agent. To apply this on a windows OS this tutorial was followed:

https://www.youtube.com/watch?v=OplGrY74qog

`ssh-add <path-to-private-key>`

Confirm the key has been added with the command below, you should see the name of your key:

`ssh-add -l`

## Ansible Development

In your `ansible-config-mgt` GitHub repository, create a new branch that will be used for development of a new feature:

![](./images/11_create_new_branch.png)

Checkout the newly created feature branch to local machine and start building code and directory structure by:

Creating directory and name it `playbooks` with a new file called `common.yml` to be first playbook

Creating a directory and name it `invertory` to keep hosts organised. Within that directory, create inventory files: `dev.yml`, `staging.yml`, `uat.yml`, `prod.yml`

![](./images/13_create_yml_files.png)

Update inventory/dev.yml file with code:
```
[nfs]
<NFS-Server-Private-IP-Address> ansible_ssh_user='ec2-user'

[webservers]
<Web-Server1-Private-IP-Address> ansible_ssh_user='ec2-user'
<Web-Server2-Private-IP-Address> ansible_ssh_user='ec2-user'

[db]
<Database-Private-IP-Address> ansible_ssh_user='ec2-user' 

[lb]
<Load-Balancer-Private-IP-Address> ansible_ssh_user='ubuntu'

```

![](./images/19_update_dev_yml.png)

In common.yml playbook configuration for repeatable, re-usable, and multi-machine tasks that is common to systems within the infrastructure will be written. Update playbooks/common.yml with code:

```
---
- name: update web, nfs and db servers
  hosts: webservers, nfs, db
  remote_user: ec2-user
  become: yes
  become_user: root
  tasks:
    - name: ensure wireshark is at the latest version
      yum:
        name: wireshark
        state: latest

- name: update LB server
  hosts: lb
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
    - name: Update apt repo
      apt: 
        update_cache: yes

    - name: ensure wireshark is at the latest version
      apt:
        name: wireshark
        state: latest
  ```
![](./images/20_update_common_yml.png)

## Commit Code To Github

Use git commands to add, commit and push your branch to GitHub. However in this project, vscode was used to perform those commands:

![](./images/21_commit_using_vscode.png)

Create a Pull request for branch:

![](./images/22_create_pull_request.png)

Merge those changes into the main branch:

![](./images/23_merge_into_main.png)

Once code changes appear in master branch – Jenkins will do its job and save all the files (build artifacts) to /var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/ directory on Jenkins-Ansible server:

![](./images/24_confirm_latest_build_has_the_changes2.png)
![](./images/24_confirm_latest_build_has_the_changes1.png)

Now run first Ansible test after cloning repo into Jenkins-Ansible server and pulling latest changes:

`cd ansible-config-mgt`
`ansible-playbook -i inventory/dev.yml playbooks/common.yml`

![](./images/25_running_playbook.png)

As shown in the image above, the db server was unreachable. After some digging, the issue was that the db server was ubuntu and did not match the RH configuration in dev.yml and common.yml. Therefore after correcting that, the image below shows ots successful:

![](./images/26_successful_playbook.png)

To verify, go to each of the servers and check if wireshark has been installed with `which wireshark` or `wireshark --version`:

![](./images/27_confim_wireshark_version_on_db.png)
