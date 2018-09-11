# Libre ROC-RK3328-CC (Renegade) - Jenkins CI/CD on an SBC

In this project, I will be running Jenkins on a single board computer.  [The Renegade](https://libre.computer/products/boards/roc-rk3328-cc/) has a bit more power than a Raspberry pi 3B+ and is handling Jenkins well enough.  I use Ansible to bootstrap Jenkins and from there Jenkins will take over all configuration, build, and deployment tasks for itself and a cluster of small machines.  It will act as the central config management node with the use of Ansible, the Ansible Gitlab plugins, and ssh access to the other hosts.  It will also act as a manager in a Docker Swarm cluster of 5 nodes and be the build server for arm32v7 and aarch64 docker images and the director of all docker swarm services.

Please refer to the source code of this project for completed references to Ansible playbooks, Jenkins Pipelines, Docker builds, and stack deploy files.
* [https://gitlab.com/jahrik/arm-jenkins](https://gitlab.com/jahrik/arm-jenkins)

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

After inserting the SD card and powering up the Renegade, it will try and receive an IP address from a DHCP server.  Once obtained, a connection can be established with the default user `root` and password `1234`.
    
    ssh root@renegade

After connecting a prompt will ask to reset the default password and create a new system user.  Give this user a password as well.  Give the new user passwordless sudo to make ansible runs easier by creating a file in `/etc/sudoers.d/your_user`.

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

Just to be clever, it's possible to have Ansible cat the Admin password as a debug message on the initial install with something like the following.

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

This node is then added to a pre-existing Docker Swarm cluster to act as the primary build and deploy node.  From any of the other 4 managers, a docker swarm token can be obtained.

    root@ninja:~# docker swarm join-token manager
    To add a manager to this swarm, run the following command:

        docker swarm join --token SWMTKN-1-token_number 192.168.123.123:2377

The above command is then entered on the jenkins node `rocks` to add it to the cluster, which then grows to 5 nodes.

    root@ninja:~# docker node ls
    ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
    ksrj43ti2is3zy4ikn13uj25w     bebop               Ready               Active              Reachable           18.06.1-ce
    rrwr1vua2496kca2w7wpubvgv *   ninja               Ready               Active              Reachable           18.06.1-ce
    n0vb407w25cdnql1jz7fi72k4     oroku               Ready               Active              Reachable           18.06.1-ce
    o8494x1qd7tiyv21nzcs4d6em     rocks               Ready               Active              Reachable           18.06.1-ce
    j9pa7cc54a0ulvmn1uahs5w59     venus               Ready               Active              Leader              18.06.1-ce

With the node added, docker commands can now be added the jenkins Pipelines.  Jenkins can now call `docker stack deploy` commands to deploy any and all stacks to the swarm cluster.

Add the jenkins user to the docker group to allow it to use docker commands without sudo

    sudo usermod -aG docker jenkins

Logging out and back in, shows that jenkins is now part of the docker group

    jenkins@rocks:/root$ groups
    jenkins docker

## Gluster

* [https://gitlab.com/jahrik/arm-gluster](https://gitlab.com/jahrik/arm-gluster)

The Odroids each have a 220G SSD drives connected and are configured to create a total of 3 replicas of any file written to a shared mount using gluster.  This project was already created beforehand and will now be added to Jenkins in the same way the above jenkins-ansible project was added.  It will also poll SCM every 5 minutes.  If I decide to make any changes to the gluster configs, I just have to push them to source control and Jenkins will handle the rest.

![jenkins_gluster.png](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/jenkins_gluster.png)

## Mining Magi Coin

* [https://gitlab.com/jahrik/arm-m-minerd](https://gitlab.com/jahrik/arm-m-minerd)

As a final example, I will stress test the Odroids by mining cryptocurrency. The Jenkins node `rocks` will build the docker image from a Dockerfile, push it up to Dockerhub, and deploy the miner service to Docker Swarm.  It will only run on nodes that are labeled `miner=true` to keep it from running on anything but the Odroids.  Any host level configuration, directory creation for volumes, etc should be handled by Ansible before deploying the service to Swarm, if mounted volumes are required.

This project starts with the [arm32v7/ubuntu](https://hub.docker.com/r/arm32v7/ubuntu/) base image.  All dependencies are installed.  It then clones the [m-cpuminer-v2](https://github.com/m-pays/m-cpuminer-v2) software, configures it, makes installs it, and preps it for execution.

*[Dockerfile](https://gitlab.com/jahrik/arm-m-minerd/blob/master/Dockerfile)*

    FROM arm32v7/ubuntu

    RUN apt-get update
    RUN apt-get install -y \
        git \
        gcc \
        make \
        automake \
        libgmp-dev \
        libcurl4-openssl-dev

    ARG workdir=/tmp
    WORKDIR $workdir
    RUN git clone https://github.com/m-pays/m-cpuminer-v2.git
    WORKDIR $workdir/m-cpuminer-v2
    RUN ./autogen.sh && ./configure CFLAGS="-O3" CXXFLAGS="-O3"
    RUN make && make install
    RUN rm -rf $workdir/m-cpuminer-v2

    CMD ["m-minerd","--benchmark"]

A simple Makefile will save repetitive command execution while building, pushing, and deploying the docker image, as well as make it a bit easier to call in the Jenkins Pipeline.

*[Makefile](https://gitlab.com/jahrik/arm-m-minerd/blob/master/Makefile)*

    IMAGE = "jahrik/m-minerd"
    TAG = "arm32v7"

    all: build

    build:
      @docker build -t ${IMAGE}:$(TAG) .
      @docker tag ${IMAGE}:$(TAG) ${IMAGE}:latest

    push:
      @docker push ${IMAGE}:$(TAG)
      @docker push ${IMAGE}:latest

    deploy:
      @docker stack deploy -c minerd-stack.yml mine

    .PHONY: all build push deploy

A Jenkins Pipeline will then tie these together.

*[Jenkinsfile](https://gitlab.com/jahrik/arm-m-minerd/blob/master/Jenkinsfile)*

    #!/usr/bin/env groovy
    node('master') {

        try {

            stage('build') {
                // Clean workspace
                deleteDir()
                // Checkout the app at the given commit sha from the webhook
                checkout scm
                sh "make"
            }

            stage('test') {
                // Run any testing suites
            }

            stage('push') {
                // Push to Dockerhub
                sh "make push"
            }

            stage('deploy') {
                // Deploy to Swarm
                sh "make deploy"
            }

        } catch(error) {
            throw error

        } finally {
            // Any cleanup operations needed, whether we hit an error or not

        }
    }

When kicked off, it will then build the docker image.

![jenkins_docker_build.png](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/jenkins_docker_build.png)

Which in turn will download and `make && make install` the m-cpuminer-v2 software.

![jenkins_make.png](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/jenkins_make.png)

I want to keep an eye on CPU temperatures as I stress test these, so I don't let it get too hot.  In order to do so, I'm going to use a [python script](https://gitlab.com/jahrik/arm-m-minerd/blob/master/templates/temp.j2) I found on the internet and write it to all three Odroid nodes with a simple Ansible [playbook](https://gitlab.com/jahrik/arm-m-minerd/blob/master/playbook.yml).  I'll then have Jenkins test against this script for a while after the miners are running and kill the service if things get too hot.
