---

layout: post
title: K8s Deployment with Ansible
date: 2043-12-05 12:00:00 -500
categories: [redhat,ansible,kubernetes,cicd]
tags: [redhat,ansible,kubernetes,cicd]

---

# Goal For This Post
Hello everyone! Today I'm back and I'm going to walkthrough how I create and spin up my Kubernetes environment at home. This is a great way to get hands on with various different technologies and allows you to start deploying applications on a local Kubernetes Environment. Because Kubernetes can be complicated and tedious to deploy (especially in enterprise environments) it would be best to automate the process of deploying and installing Kubernetes automatically through the use of an Infrastructure as Code approach. My personal preference for this would be using Ansible. With all of that in mind, the goal of this post is to walk you through the process of deploying Kubernetes on local bare-metal using an automated ansible deployment!

# Prepping the Environment
To begin, it's important to understand that you don't necessarily need to use bare-metal for this setup. If you have a beefy-enough server and can run a couple of virtual machines, that would also suffice. Or you could even use a cloud platform like AWS/GCP and deploy EC2 instances! In this setup however I'm going to be deploying Kubernetes on 5 nodes, having 1 Master and 4 worker nodes. It's actually advisable to deploy Kubernetes while having an odd number of Worker nodes, but I'm not going to be running any heavy workload on them and want the extra resource efficiency gained by the extra worker node. 

All of my nodes will be installed using a RedHat 9.5 image, and I will have a separate laptop that I will designate as my ansible-controller. My laptop will ultimately be where I deploy the cluster using Kubernetes and ultimately manage it. My laptop is running a flavor of Rocky Linux. 

Assuming you have Rhel9 installed on all of your nodes, lets first start by installing ansible on your controller:
```bash
dnf install -y ansible
```

Then lets create a directory to work out of:
```bash
mkdir -pv /home/user/Kube_Ansible
```

Now let's create a hosts file. This file is what ansible uses to know which hosts to connect to and ultimately make changes. Because I already installed an OS on my hosts, I also configured the hostnames and DNS; therefore I update my DNS server to reflect their hostnames. All that to say, I configured my hosts file to use DNS records, however if you didn't want to setup DNS you could specify each IP. My hosts file is also grouped by what host I want to be my Master/Worker and is denoted through `[ ]` square brackets. 
```txt
# /etc/ansible/hosts
[Manager]
mainrhel

[Workers]
rhel9bee
gmrhel
acerhel
mainrhelb
```

An optional step is to copy this directory into the following directory (or just edit that file directly) so that you won't have to specify a hosts file when running ansible.
```bash
mv hosts /etc/ansible/hosts
```

We will want to use public/private key infrastructure when authenticating through ansible, therefore I will create and push my laptops host root ssh keys using the following command:
```bash
ssh-keygen
```
![collector][/assets/k8s/k81.png]

There are 2 more things we will need to do before running our first ansible script; first we will need to make sure that host_key_checking is disable in the main ansible configuration file. If this setting is not set; then when we try to ssh into our nodes the first time we will get an error asking if we are sure that the ssh host is valid and to type yes to continue the connection.
```bash
export ANSIBLE_HOST_KEY_CHECKING=False
```
Second, we will need to install some ansible packages. Ansible packages allow Ansible (and really python) more options when specifying commands. We will be installing the posix collection which has a bunch of built-ins for managing Linux systems and the community.general collection to manage modules inside our Rhel9 systems.
```bash
ansible-galaxy collection install ansible.posix
ansible-galaxy collection install community.general
```

Now let's create our first playbook, this will seem a little complicated if you don't have much experience writing playbooks or don't work with yaml files a lot, the important thing to understand is that it will copy the public key in this directory and put it in the appropriate directory on all the nodes so that we can ssh into the nodes in the future.
```yaml
# push_pubkey.yaml
- name: Deploy my root public key to all nodes for passwordless SSH
  hosts: all
  become: true
  become_method: sudo

  tasks:
    - name: Ensure the .ssh directory exists
      ansible.builtin.file:
        path: /root/.ssh
        state: directory
        mode: 0700

    - name: Deploy my root public key
      ansible.posix.authorized_key:
        user: root
        key: "{{ lookup('file', '/root/.ssh/id_rsa.pub') }}"
        state: present
```

You will want to run this playbook with the following command; note that you will be prompted to enter the root password for all of these hosts, it would be a good idea after you push the keys to go back and disable username/password ssh login for root (maybe even through ansible 👀). Also note that your output will be slightly different than mine, as I already successfully pushed my keys, but the end result should stay the same.
```bash
ansible-playbook --ask-pass push_pubkey.yaml
```
![collector][/assets/k8s/k82.png]

# Opening the firewall and other networking
Networking can be a pretty daunting concept to understand and unfortunately that doesn't stop with Kubernetes. Fortunately, now that we understand an can run ansible playbooks there is another playbook that we can run that will open all of the necessary ports in order for the various Kubernetes APIs to communicate with one another. You'll notice that these are seperated by the role of the node (Manager v. Worker). Additionally at the bottom of this playbook I also enabled the ability to forward traffic. This is essential to K8s working and thus is required and the reason it's incorporated in this playbook:
```bash
ansible-playbook firewall.yaml
```
```yaml
---
- name: Open Up Required Ports
  hosts: Manager
  tasks: 
    - name: Open ports for Master Node
      vars:
        ports_to_open:
          - 179/tcp
          - 6443/tcp
          - 2379-2380/tcp
          - 10250/tcp
          - 10259/tcp
          - 10257/tcp
      ansible.posix.firewalld:
        port: "{{ item }}"
        permanent: true
        state: enabled
        immediate: yes
      loop: "{{ ports_to_open }}"

- name: Open Ports for Worker Nodes
  hosts: Workers 
  tasks: 
    - name: Open Ports for Worker Nodes
      vars:
        worker_ports:
          - 179/tcp
          - 10250/tcp
          - 10256/tcp
          - 30000-32767/tcp
      ansible.posix.firewalld:
        port: "{{ item }}"
        permanent: true
        state: enabled
        immediate: yes
      loop: "{{ worker_ports }}"

- name: Enable port forwarding
  hosts: all
  tasks:
    - name: Load br_netfilter kernel module
      modprobe:
        name: br_netfilter
        state: present

    - name: Enable bridge netfilter for iptables
      sysctl:
        name: net.bridge.bridge-nf-call-iptables
        value: 1
        state: present
        reload: yes

    - name: Enable IPv4 forwarding
      sysctl:
        name: net.ipv4.ip_forward
        value: 1
        state: present
        reload: yes
```
![collector][/assets/k8s/k83.png]

We are almost ready to begin, but before we start the automated deployment we are going to make sure that all of our systems are registered and updated. If you don't already you can sign up for a RedHat developer license in order to deploy these hosts for free. **Make sure you change the values for your username/password**
```bash
ansible-playbook dnf_install.yaml
```
```yaml
---
# Update Username and Password
- name: Ensure system is registerd
  hosts: all
  tasks:
    - name: register systems
      community.general.redhat_subscription:
        username: "username"
        password: "password"

- name: RHEL DNF Install Updates
  hosts: all
  tasks:
    - name: update systems
      ansible.builtin.dnf:
         name: "*"
         state: latest
```

# Understanding the Deployment
Finally, the moment of truth! We are ready to deploy and if you're ready to go then please feel free to scroll down and launch; otherwise I'm going to spend a couple of paragraphs talking about what the script is doing in order for this deployment to be successful and some of the pitfalls of running k8s on Rhel.

## Disabling Swap
The first part of the script aims to disable swap on all nodes in the cluster. With swap enable, Kubernetes can't easily determine performance and degredation of resources on each worker node so it is recommended to disable swap. This is a relatively easy fix, and with 2 tasks of turning the swap off first and then making the change persistent by editing /etc/fstab.
![collector][/assets/k8s/k84.png]

## Disabling SElinux
Now this is where the deployment gets a little scary. If you're not aware, SElinux is a security architecture for Linux systems that allow control over who/what has access to the systems. It's also notorious for causing issues when rolling out new software and this is definitely the case with K8s.Kubernetes recommends disabling SElinux as there is a lot of processes and kernel calls that various processes/containers/services that are needed to successfully deploy. With that in mind, disabling SElinux could expose your system to more risk, so please proceed at your own caution.

```yaml
- name: Disable SElinux and Add Kube Repo
  hosts: all
  tasks:
  - name: Set SELinux to permissive mode temporarily
    command: setenforce 0
    register: result2
    changed_when:
      - result2.rc == 0

  - name: Disable SELinux permanently
    command: sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
    register: result3
    changed_when:
      - result3.rc == 0
```

## Installing a Container Runtime
Contrary to what you may already believe, Kubernetes (and its associated APIs) aren't actually the service deploying containers, that is handled by the container runtime. Kubernetes also doesn't ship with a standard container runtime and instead posts guidelines that vendors need to meet in order to be complient. RedHat has a kubernetes service called Openshift that uses the CRI-O runtime, however that runtime is only available if you have the appropriate registries (through licensing). Hence a workaround is required; and thus I decided to download and implement containerd as the container runtime in this k8s deployment. This section of the ansible playbook involves downloading the right binaries and installing them on all systems. This also includes the CNI binaries that are necessary for Kubernetes services.
![collector][/assets/k8s/k85.png.png]

## Kubelet, Kubeadm, Kubectl
This is where the real meat and patatoes of the playbook come into play. The importtant thing to note about this section of the playbook is that all the dependenies will be manually installed like conntrack on all the nodes. All of these services will be installed but they won't be initalized yet. That will ultimately be the job of Kubeadm and configured on the manager node. There is one variable that you may want to change, which is the pod_cidr. In this configuration my private home network is in the 192.168.1.0/24 subnet, Kubernetes needs it's own dedicated pod network which is why I specified in the kubeadm deployment the 10.0.0.0/16, so if your home network is in the 10.0.0.0/16 subnet you will want to change it. It's also important to know that this script is written in a way in which you can run this script multiple times. Once you deploy kubeadm you can't re-run the configuration, however this script is written that it will skip that process if you want to change something and re-run it. 

## Installing a CNI
Once Kubadm is installed, we have to configure the networking. Thankfully we can have software called a container network interface (CNI) handle the routing and switching inside the cluster. Again the script manually installs the CNI, configures it based off our current configuration and apply it. The CNI will now show up as a pod running inside the cluster. The important thing to note is if you changed your pod_cidr network from earlier you will also want to change it here too.
![collector][/assets/k8s/k86.png.png]