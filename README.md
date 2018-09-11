# Libre ROC-RK3328-CC (Renegade) - Jenkins CI/CD on an SBC

In this project, I will be running Jenkins on a single board computer.  [The Renegade](https://libre.computer/products/boards/roc-rk3328-cc/) has a bit more power than a Raspberry pi 3B+ and is handling Jenkins well enough.  I use Ansible to bootstrap Jenkins and from there Jenkins will take over all configuration, build, and deployment tasks for itself and a cluster of small machines.  It will act as the central config management node with the use of Ansible, the Ansible plugin, and ssh access to the other hosts.  It will also act as a manager in a Docker Swarm cluster of 5 nodes and be the build server for arm32v7 and aarch64 docker images and the director of all docker swarm services.

![renegade_front_right](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/renegade_front_right.jpg)

## Hardware
* [Libre Computer Board ROC-RK3328-CC (Renegade)](https://www.amazon.com/gp/product/B078RT6H8X/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)
* [Libre Computer Board Heatsink for ROC-RK3328-CC](https://www.amazon.com/gp/product/B0792VXBVH/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)
* [SanDisk Ultra 32GB microSD](https://www.amazon.com/gp/product/B010Q57T02/ref=oh_aui_detailpage_o01_s00?ie=UTF8&psc=1)

Not included in this build, but a main part of what this Jenkins node will be configuring and controlling is a previously built [3 node Odroid cluster](https://homelab.business/odroid-hc-1-cluster-build/) that are all Docker Swarm managers and will run most of the Docker Swarm services.  They use gluster to provide replicated storage to the Swarm.  It also includes  a raspberry pi 2b, that runs pihole and is also a manager in the Docker Swarm cluster.

![renegade_front_left.jpg](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/renegade_front_left.jpg)

## OS install

Install ubuntu 18.04 on the Renegade from the Armbian project repos
* https://www.armbian.com/renegade/

Flash the SD card with dd

    7z e Armbian_5.59_Renegade_Ubuntu_bionic_default_4.4.152_desktop.7z
    sudo dd if=Armbian_5.59_Renegade_Ubuntu_bionic_default_4.4.152_desktop.img of=/dev/mmcblk0

After inserting the SD card and powering up the Renegade, it will try and obtain a IP address from a DHCP server.  Once obtained a connection can be established with
    
    ssh root@renegade

After connecting a prompt will reset the default password, `1234` and create a new system user.  Give this user a password as well.  Give the new user passwordless sudo to make ansible runs easier by creating a file in `/etc/sudoers.d/your_user`

    #/etc/sudoers.d/your_user*
    your_user ALL=(ALL) NOPASSWD: ALL

Ensure the system is up to date

    apt-get update
    apt-get upgrade -y

Generate an ssh key that will be used to connect to the other hosts

    ssh-keygen -b 4096 -t rsa -f ~/.ssh/id_rsa -C "ansible user"

Update the hostname in `/etc/hostname` & `/etc/hosts`
I'm choosing `rocks`

    #/etc/hostname 
    rocks

    #/etc/hosts
    127.0.0.1   localhost rocks
    ::1         localhost rocks ip6-localhost ip6-loopback
    ...
    ...

Configure Timezone

    dpkg-reconfigure tzdata 
    ...
    ...
    Current default time zone: 'America/Los_Angeles'
    Local time is now:      Sun Sep  9 20:22:27 PDT 2018.
    Universal Time is now:  Mon Sep 10 03:22:27 UTC 2018.

## Ansible Install

Ensure python is installed

    sudo apt-get install python

Add the Ansible repo and install

    sudo apt-get install software-properties-common
    sudo apt-add-repository ppa:ansible/ansible
    sudo apt-get update
    sudo apt-get install ansible

The above manual installation can be accomplished with the following Ansible playbook, which will be included in the first Jenkins Pipeline created.

*[ansible_install.yml](https://gitlab.com/jahrik/arm-jenkins/blob/master/ansible_install.yml)*

    - hosts: ansible
      become: true
      become_method: sudo

      vars:

        ansible:
          repo: ppa:ansible/ansible

      tasks:

      - name: Install dependencies
        apt:
          name: "{{ item }}"
          state: present
          update_cache: yes
        with_items:
          - python
          - software-properties-common

      - apt_repository:
          repo: "{{ ansible.repo }}"
          state: present

      - name: Install Ansible
        apt:
          name: ansible
          state: present
          update_cache: yes

## Jenkins Install

Jenkins is just as simple to install.

First Java needs to be installed

    sudo apt-get install openjdk-8-jre

Installation steps taken from [jenkins.io/doc](https://jenkins.io/doc/book/installing/)

    wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
    sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
    sudo apt-get update
    sudo apt-get install jenkins

Converting this to Ansible tasks looks like the following

*[jenkins_install.yml](https://gitlab.com/jahrik/arm-jenkins/blob/master/jenkins_install.yml)*

    - hosts: jenkins
      become: true
      become_method: sudo

      vars:

        jenkins:
          key_url: https://pkg.jenkins.io/debian/jenkins.io.key
          repo: deb http://pkg.jenkins.io/debian-stable binary/

      tasks:

        - name: Install java
          apt:
            name: openjdk-8-jre
            state: present

        - name: Add apt signing key for Jenkins
          apt_key:
            url: "{{ jenkins.key_url }}"
            state: present

        - name: Add apt repository for Jenkins
          apt_repository:
            repo: "{{ jenkins.repo }}"
            state: present

        - name: Install Jenkins
          apt:
            name: jenkins
            state: present
            update_cache: yes

Just to be clever, it's possible to have Ansible cat the Admin password as a debug message on installation with something like the following.

        - name: Cat password to debug
          debug:
            msg: /var/lib/jenkins/secrets/initialAdminPassword

        - name: Cat admin pass
          command: "cat /var/lib/jenkins/secrets/initialAdminPassword"
          register: admin_pass

        - name: Display admin pass
          debug: msg={{ admin_pass.stdout }}
          when: admin_pass

Navigate to `your_host:8080` on the jenkins node and login to configure the jenkins user, passwords, etc... Checkout the jenkins.io [getting-started](https://jenkins.io/doc/pipeline/tour/getting-started/) docs for further configuration.

## Jenkins Plugins

With Jenkins and Ansible installed, use Jenkins to run all subsequent Ansible playbooks from now on to keep configuring itself and all other hosts.  A few plugins need to be installed, first.
* [Ansible plugin](https://wiki.jenkins.io/display/JENKINS/Ansible+Plugin)
* [AnsiColor](https://wiki.jenkins.io/display/JENKINS/AnsiColor+Plugin)

Use the GitLab plugin to poll for SCM changes every 5 minutes.  The same can be accomplished with the Github and Bitbucket plugins.
* [GitLab plugin](https://wiki.jenkins.io/display/JENKINS/GitLab+Plugin)

## GitLab

Create an [API token](https://gitlab.com/profile/personal_access_tokens) on GitLab to connect the Jenkins plugin.

![gitlab_token.png](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/gitlab_token.png)

## Jenkins Credentials

Navigate to `Jenkins > Credentials > System > Global Credentials` and Create a new GitLab Token credential.

![jenkins_gitlab_token_01.png](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/jenkins_gitlab_token_01.png)
![jenkins_gitlab_token_02.png](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/jenkins_gitlab_token_02.png)

Also, add the ssh key generated for the ansible.

![jenkins_ansible_key.png](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/jenkins_ansible_key.png)

## Jenkins Pipeline Project

Create a new Pipeline project `ansible-jenkins`

![ansible_jenkins.png](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/ansible_jenkins.png)

I chose to keep 3 days worth of build history with a max of 5 builds to keep.

![log_rotate.png](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/log_rotate.png)

Configure the project to build on a push event to GitLab and to poll SCM every 5 minutes.

![jenkins_build_on_push.png](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/jenkins_build_on_push.png)

Lastly, choose a `Pipeline script from SCM`, enter the clone url of the project, the branch to follow, and the Name of the `Jenkinsfile`.

![jenkins_pipeline_configs.png](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/jenkins_pipeline_configs.png)

Save the project and start building!  Changes pushed to the master branch of GitLab will kick off a playbook including all above configurations that were made plus everything that needs configured form here on out.  As playbooks are added to the pipeline and pushed up to Github, Jenkins will poll every 5 minutes, see these changes and deploy the Pipeline again and again, automatically.

![jenkins_pipeline_configs.png](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/jenkins_pipeline_configs.png)

A basic Ansible Pipeline.

* [inventory.ini](https://gitlab.com/jahrik/arm-jenkins/blob/master/inventory.ini)
* [playbook.yml](https://gitlab.com/jahrik/arm-jenkins/blob/master/playbook.yml)

*[Jenkinsfile](https://gitlab.com/jahrik/arm-jenkins/blob/master/Jenkinsfile)*

    #!/usr/bin/env groovy
    node('master') {

        try {

            stage('build') {
                // Clean workspace
                deleteDir()
                // Checkout the app at the given commit sha from the webhook
                checkout scm
            }

            stage('test') {
                // Run any testing suites
                sh "echo 'WE ARE TESTING'"
            }

            stage('deploy') {
                sh "echo 'WE ARE DEPLOYING'"
                ansiColor('xterm') {
                    ansiblePlaybook(
                        playbook: 'playbook.yml',
                        inventory: 'inventory.ini',
                        // limit: 'local',
                        colorized: true)
                }
            }

        } catch(error) {
            throw error

        } finally {
            // Any cleanup operations needed, whether we hit an error or not

        }
    }

## Hosts

* rocks
* bebop
* venus
* ninja
* oroku

The other hosts in the inventory file are the 3 Odroids and a raspberry pi 2B.  They are all already managers in a Docker Swarm cluster of 4, of which the Renegade `rocks` will be added.  Jenkins will take over configurations and deployments I have been doing up to this point from my laptop.  First, they will each need a new jenkins user with ssh and sudo access.

To each host in the cluster, add a jenkins user, create jenkins group, create a password.

    adduser jenkins

And grant jenkins passwordless sudo to each host ansible will connect to by creating a new file at `/etc/sudoers.d/jenkins`

    #/etc/sudoers.d/jenkins
    jenkins ALL=(ALL) NOPASSWD: ALL

Then, from the Jenkins host and as the jenkins user, add the ssh key to all other hosts in the cluster including itself.

    ssh rocks
    su jenkins
    ssh-copy-id rocks
    ssh-copy-id bebop
    ssh-copy-id venus
    ssh-copy-id ninja
    ssh-copy-id oroku

Once connectivity and sudo access have been established, it can be tested by hitting all hosts with the Ansible ping module.

    cd /var/lib/jenkins/workspace/ansible-jenkins
    jenkins@rocks:~/workspace/ansible-jenkins$ ansible -i inventory.ini all -m ping
    venus | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    rocks | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    ninja | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    bebop | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    oroku | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }

## Docker

This node is then added to a pre-existing Docker Swarm cluster to act as the primary build and deploy node.  From any of the other 4 managers, a docker swarm token is obtained.

    root@ninja:~# docker swarm join-token manager
    To add a manager to this swarm, run the following command:

        docker swarm join --token SWMTKN-1-352mfchgq520dgrf7u1f7jr78703pbcotcxuh127rjbay1pp80-6wz38c4uwlq2crktamnogngpj 192.168.2.241:2377

The above command is then entered on the jenkins node to add it to the cluster, which then grows to 5 nodes.

    root@ninja:~# docker node ls
    ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
    ksrj43zy4ikn13u3ti2isj25w     bebop               Ready               Active              Reachable           18.06.1-ce
    vua2496krrwr1ca2w7wpubvgv *   ninja               Ready               Active              Reachable           18.06.1-ce
    n0vb407wdnql1jz7f25ci72k4     oroku               Ready               Active              Reachable           18.06.1-ce
    o8494d7tiyv21x1qnzcs4d6em     rocks               Ready               Active              Reachable           18.06.1-ce
    j9pa4a0ulvmn17cc5uahs5w59     venus               Ready               Active              Leader              18.06.1-ce

## Gluster

## Magi Coin miners on Docker Swarm
