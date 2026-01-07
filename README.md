# pi-cluster

[![CI](https://github.com/steffkelsey/pi-cluster/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/steffkelsey/pi-cluster/actions/workflows/ci.yml)

An Ansible playbook to install the software needed to run kubernetes on a cluster
of Raspberry Pis.

Currently, my cluster is one Raspberry Pi 400 and one pi 4B but the plan is to get more
sophisticated and add more SBCs including Rockchip.

This repo gives instructions on how to setup the hardware and stops after the 
basic software is installed. The actual software that will run on k8s in the
cluster is managed by ArgoCD and will be in a repo [here](https://github.com/steffkelsey/homelab-live).

- [x] k3s cluster control_plane
- [x] k3s cluster node

additional packages  
 - [x] Tailscale  
 - [x] k8s Secret for Tailscale k8s-operator oauth client  
 - [x] ArgoCD  
 - [x] helm (needed just for installing of argoCD) 
 - [x] kubernetes client for python (needed just for installing other stuff) 
 - [x] nfs-kernel-server for serving files in a USB Mount (temporary until new more energy efficient network attached storage is setup)  
 - [ ] for multinode clusters, disable ServiceLB at k3s server installed
 (`--disable=servicelb`) and install Cilium LoadBalancer via ArgoCD in the homelab-live repo
 - [ ] disable flannel (`--flannel-backend=none --disable-network-policy`) at k3s server install in favor of more secure CNI (Cilium)
 - [ ] install Cilium CLI on the server node

## Setup Raspberry Pi

I decided to provision each pi with Ubuntu Server 24.04 (rather than the lighter
 Raspberry Pi OS) because it is better representative of what I would be running
in production if I was running k8s in the cloud.

I flashed Ubuntu Server 24 to the SD Card using the Raspberry Pi Imager.

To make network discovery and integration easier, I edit the advanced
configuration in Imager, and set the following options:

  - Set hostname: `node1.local` (set to `2` for node 2, `3` for node 3, etc.)
  - Enable SSH: 'Allow public-key', and paste in my public SSH key(s)

- [x] set static ip on each Pi

On my workstation where I will be running Ansible to provision each Pi, I edited
the ~/.ssh/config file for each Host.

```
Host node1.local
  Hostname 192.168.xx.xx
  Port 22
  User steff
```

Once avahi is up and runing (and publish-workstation=yes), then the ssh config
will no longer matter and you can ssh by `ssh steff@node1.local` as long as 
you are on a network that can see node1.local with `avahi-browse`.

## Setup Ansible

First, install Molecule, Ansible and the linters in a virtual env (not on a Pi,
on the computer you will use to SSH into the each Pi in the cluster).

```bash
# Setup using a virtual env
python3 -m venv .venv
# activate the env
source .venv/bin/activate
# (optional) update pip to the latest
pip3 install --upgrade pip
# Install everything from the requirements file
pip3 install -r requirements.txt

```

Second, install the dependencies for the main playbook and the roles.
```bash
ansible-galaxy install -r requirements.yml
```
in this directory


## SSH Connection Test

To be sure that Ansible can connect to each node, test with:
```bash
ansible all -m ping
```


## Running

To provision all the roles in this repo for the entire pi-cluster in hosts.ini, navigate
to the root directory of this repo and run:  
```bash
ansible-playbook main.yml --ask-become-pass
```

To upgrade all the software installed with apt, run   
```bash
ansible-playbook upgrade.yml --ask-become-pass
```

## Tests

To test the main playbook, go to the top level directory and run:  
```bash
molecule test
```

## FYI 
I had trouble testing the playbook with molecule from a host running Rocky
Linux 9. The issue is with older docker containers needing the host to have
ip_tables present. Info
[here](https://ryandaniels.ca/blog/docker-and-trouble-with-red-hat-enterprise-linux-9-iptables/).

To see the error, login to a running container and try `dockerd`. You should
see an error about ip_tables not being present. In that case, use the fix from
the link above.

## Debugging

To debug the playbook, from the project root:  

1. start the dev environment `$ molecule converge`  
2. ssh into the container `$ molecule login`  
3. rerun the playbook `$ molecule converge`  
4. run the linter `$ molecule lint`  
5. run the verify steps `$ molecule verify`  
6. clean up `$ molecule destroy`  
7. testing - see the Tests section above for running the entire cycle which
includes creating, verifying, and destroying the environment

### Kubernetes Debugging within Molecule containers
Connecting to services running on k3s within a docker container managed by
Ansible Molecule is a bit tricky. Below is a runbook for connecting with ArgoCD
UI:  

```bash
# Log into the container
molecule login

# Find the argoCD admin password
kubectl get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" --namespace=argocd | base64 --decode

# port forward to the argocd server - must use --address=0.0.0.0 for the container
# wrapping k3s to listen on all incoming IPs. The default of 127.0.0.1 will
# only work inside the container running k3s and not from the host. We have a 
# host within a host.
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address=0.0.0.0

# see the contents of a kubernetes secret
kubectl get secret <secret-name> -o json | jq '.data | map_values(@base64d)'

```

Now, you can find the UI by typing https://localhost:8080 into the browser
