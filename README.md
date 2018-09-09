# Libre ROC-RK3328-CC (Renegade) - Jenkins CI/CD on an SBC

In this project, I will be installing Jenkins on a single board computer.  The Renegade has a bit more power than a Raspberry pi 3B+ and is handling Jenkins well enough.  I use Ansible to bootstrap Jenkins and from there Jenkins will take over all configuration, build, and deployment tasks for a cluster of small machines.  It will act as the central config management node with the use of Ansible, the Ansible plugin, and ssh access to the other hosts.  It will also act as a manager in a Docker Swarm cluster of 5 nodes and be the build server for arm32v7 and aarch64 docker images and the director of all docker swarm services.

![renegade_front_right](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/renegade_front_right.jpg)

## Hardware
* [Libre Computer Board ROC-RK3328-CC (Renegade)](https://www.amazon.com/gp/product/B078RT6H8X/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)
* [Libre Computer Board Heatsink for ROC-RK3328-CC](https://www.amazon.com/gp/product/B0792VXBVH/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)
* [SanDisk Ultra 32GB microSD](https://www.amazon.com/gp/product/B010Q57T02/ref=oh_aui_detailpage_o01_s00?ie=UTF8&psc=1)

Not included in this repo, but a main part of what this Jenkins node will be configuring and controlling is a previously built [3 node Odroid cluster](https://homelab.business/odroid-hc-1-cluster-build/) that are all Docker Swarm managers and will run most of the Docker Swarm services.  They use gluster to provide replicated storage to the Swarm.  It also includes  a raspberry pi 2b, that runs pihole and is also a manager in the Docker Swarm cluster.

![renegade_front_left.jpg](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/renegade_front_left.jpg)

## OS install

Install ubuntu 18.04 on the Renegade from the Armbian project repos
* https://www.armbian.com/renegade/

## Jenkins Install

Initialize an inventory.ini file.  My hosts are as follows:


| HOST | purpose |
| rocks | jenkins |
| bebop | pihole |
| venus | swarm,gluster |
| ninja | swarm,gluster |
| oroku | swarm,gluster |

[*inventory.ini*](https://gitlab.com/jahrik/arm-jenkins/blob/master/inventory.ini)

    [jenkins]
    rocks

    # [local]
    # rocks ansible_connection=local

    [cluster]
    bebop
    venus
    ninja
    oroku

    [docker]
    rocks
    bebop
    venus
    ninja
    oroku

### Install java

    - name: Install java8
      apt:
        name: openjdk-8-jre
        state: present
      tags:
        - java

## Jenkins Plugins
* Ansible
* AnsiColor

## Hosts

## Ansible

## Docker

- name: Install java8
  apt:
    name: openjdk-8-jre
    state: present
  tags:
    - java

- name: Add apt signing key for Jenkins
  apt_key:
    url: "{{ jenkins.key_url }}"
    state: present
  tags:
    - jenkins

- name: Add apt repository for Jenkins
  apt_repository:
    repo: "{{ jenkins.repo }}"
    state: present
  tags:
    - jenkins

- name: Install Jenkins
  apt:
    name: jenkins
    state: present
    update_cache: yes
  tags:
    - jenkins

#     - name: Cat password to debug
#       debug:
#         msg: /var/lib/jenkins/secrets/initialAdminPassword

#     - name: Cat admin pass
#       command: "cat /var/lib/jenkins/secrets/initialAdminPassword"
#       register: admin_pass

#     - name: Display admin pass
#       debug: msg={{ admin_pass.stdout }}
#       when: admin_pass
