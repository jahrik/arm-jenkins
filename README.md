# Libre ROC-RK3328-CC (Renegade) - Jenkins CI/CD on an SBC

In this project, I will be installing Jenkins on a single board computer.  The Renegade has a bit more power than a Raspberry pi 3B+ and is handling Jenkins well enough.  I use Ansible to bootstrap Jenkins and from there Jenkins will take over all configuration, build, and deployment tasks for a cluster of small machines.  It will act as the central config management node with the use of Ansible, the Ansible plugin, and ssh access to the other hosts.  It will also act as a manager in a Docker Swarm cluster of 5 nodes and be the build server for arm32v7 and aarch64 docker images and the director of all docker swarm services.

![renegade_front_right](https://gitlab.com/jahrik/arm-jenkins/raw/master/pics/renegade_front_right.jpg)

## Hardware
* [Libre Computer Board ROC-RK3328-CC (Renegade)](https://www.amazon.com/gp/product/B078RT6H8X/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)
* [Libre Computer Board Heatsink for ROC-RK3328-CC](https://www.amazon.com/gp/product/B0792VXBVH/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)
* [SanDisk Ultra 32GB microSD](https://www.amazon.com/gp/product/B010Q57T02/ref=oh_aui_detailpage_o01_s00?ie=UTF8&psc=1)

Not included in this build, but a main part of what this Jenkins node will be configuring and controlling is a previously built [3 node Odroid cluster](https://homelab.business/odroid-hc-1-cluster-build/) that are all Docker Swarm managers and will run most of the Docker Swarm services.  They use gluster to provide replicated storage to the Swarm.  It also includes  a raspberry pi 2b, that runs pihole and is also a manager in the Docker Swarm cluster.

## Jenkins Install

## Jenkins Plugins
* Ansible
* AnsiColor

## Hosts

## Ansible

## Docker
