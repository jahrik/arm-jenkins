# Libre ROC-RK3328-CC (Renegade) - Jenkins CI/CD on an SBC

In this project, I will be installing Jenkins on a low power SBC.  It has a bit more power than a Raspberry pi 3B+ and is handling the load, nicely.  I use Ansible to bootstrap this Jenkins node and then have it keep configuring itself while the Ansible playbooks continue to grow.  It will also act as the central config management node for the rest of the SBCs in the homelab.

The purpose of this machine
* Ansible config management for SBC cluster
* Docker builds for armhf
* Docker swarm deployments

## Hardware
* [Libre Computer Board ROC-RK3328-CC (Renegade)](https://www.amazon.com/gp/product/B078RT6H8X/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)
* [Libre Computer Board Heatsink for ROC-RK3328-CC](https://www.amazon.com/gp/product/B0792VXBVH/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1)
* [SanDisk Ultra 32GB microSD](https://www.amazon.com/gp/product/B010Q57T02/ref=oh_aui_detailpage_o01_s00?ie=UTF8&psc=1)

Not included in this build, but a main part of what this Jenkins node will be configuring and controlling is a previously built [3 node Odroid cluster](https://homelab.business/odroid-hc-1-cluster-build/) that are all Docker Swarm managers and will run most of the Docker Swarm services.  They use gluster to provide replicated storage to the Swarm.  It also includes  a raspberry pi 2b, that runs pihole, is a DHCP and DNS server for the homelab, and is also a manager in the Docker Swarm cluster.

## Jenkins

## Jenkins Plugins
* Ansible
* AnsiColor

## Ansible

## Docker

## Hosts

