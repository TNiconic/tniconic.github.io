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
![ssh-keygen][/assets/k8s/k81.png]

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
![push-keys][/assets/k8s/k82.png]

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
          - 5473/tcp
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
          - 5473/tcp
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
![opening_firewall][/assets/k8s/k83.png]

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
![disabling_swap][/assets/k8s/k84.png]

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
![containerd][/assets/k8s/k85.png.png]

## Kubelet, Kubeadm, Kubectl
This is where the real meat and patatoes of the playbook come into play. The importtant thing to note about this section of the playbook is that all the dependenies will be manually installed like conntrack on all the nodes. All of these services will be installed but they won't be initalized yet. That will ultimately be the job of Kubeadm and configured on the manager node. There is one variable that you may want to change, which is the pod_cidr. In this configuration my private home network is in the 192.168.1.0/24 subnet, Kubernetes needs it's own dedicated pod network which is why I specified in the kubeadm deployment the 10.0.0.0/16, so if your home network is in the 10.0.0.0/16 subnet you will want to change it. It's also important to know that this script is written in a way in which you can run this script multiple times. Once you deploy kubeadm you can't re-run the configuration, however this script is written that it will skip that process if you want to change something and re-run it. 

## Installing a CNI
Once Kubadm is installed, we have to configure the networking. Thankfully we can have software called a container network interface (CNI) handle the routing and switching inside the cluster. Again the script manually installs the CNI, configures it based off our current configuration and apply it. The CNI will now show up as a pod running inside the cluster. The important thing to note is if you changed your pod_cidr network from earlier you will also want to change it here too.
![calico][/assets/k8s/k86.png.png]

## Joining the Worker Nodes
At this point in the script, kubeadm has been successfully deployed, the networking is in place and all there is left to do is joing the worker nodes to the manager. This is a relatively simple process, first you take the output of the `kubeadm token create --print-join-command` which will print the command that the workers will use to connect back to the manager and then run it on the worker nodes. I had to play around with variables, so that the manager didn't accidentally try to join itself, so the workaround was to download the output of the above command to a file locally on the ansible-controller and then push it to the worker nodes and have them run the command from the file.
![join_nodes][/assets/k8s/k87.png]

# Configuring our Ansible-Controller to manage our K8s cluster
Congrats! At this point, assuming everything went smooth you should now have a fully functioning Kubernetes cluster ready to deploy pods (containers). Before we do some diagnostic checks, lets first configure our ansible-controller to be able to administer the cluster; preventing us from having to ssh into a node. Again here is a quick playbook that will configure the localhost to administer the cluster (assuming you're logged in as the root user):
```yaml
---
- hosts: localhost
  gather_facts: false
  tasks:
    - name: Download kubectl binary
      get_url:
        url: "https://dl.k8s.io/release/v1.27.3/bin/linux/amd64/kubectl"
        dest: /usr/local/bin/kubectl
        mode: '0755'

    - name: Verify kubectl installation
      command: kubectl version --client

    - name: Create the kubeconfig directory
      ansible.builtin.file:
        path: /root/.kube
        state: directory
        mode: 0755

    - name: Fetch kubeconfig from control plane node
      ansible.builtin.fetch:
        src: /etc/kubernetes/admin.conf
        dest: /root/.kube/config
        flat: yes
      delegate_to: mainrhel
      become: true

    - name: Set KUBECONFIG environment variable
      ansible.builtin.lineinfile:
        path: /root/.bashrc
        line: 'export KUBECONFIG=/root/.kube/config'
        create: yes

    - name: Add alias for kubectl
      ansible.builtin.lineinfile:
        path: /root/.bashrc
        line: 'alias k=kubectl'
        create: yes

    - name: Enable tab auto-completion
      shell: source <(kubectl completion bash)" >> ~/.bashrc
      args:
        executable: /bin/bash

    - name: Apply tab auto-completion
      shell: source ~/.bashrc
      args:
        executable: /bin/bash
```

# Debugging and Running our first Pod
Now that we have the ability to manage our cluster locally, let's first ensure that all of our nodes are connected and in the ready status:
```bash
kubectl get nodes
```
![get-nodes][/assets/k8s/k88.png]

Everything is looking good, now we can view all of the pods that are currently running on the system to make sure everything is functioning like it should. Remember, these pods are used by Kubernetes to keep Kuberentes (and Calico networking running)
```bash
kubectl get pods -A
```
![get-pods][/assets/k8s/k89.png]

The last demonstration before I post the full playbook is to prove that we can run a container. For this we will just run a basic nginx container and make sure it's running:
```bash
kubectl run nginx --image=nginx

kubectl describe pod nginx
```
![describe_nginx][/assets/k8s/k810.png]
```yaml
---
- name: Persistently disable swap
  hosts: all
  tasks:
    - name: Disable swap for current session
      command: swapoff -a
      register: result
      changed_when:
        - result.rc == 0

    - name: Disable Swap permanently
      replace:
        path: /etc/fstab
        regexp: '^(\s*)([^#\n]+\s+)(\w+\s+)swap(\s+.*)$'
        replace: '#\1\2\3swap\4'
        backup: yes
        
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

  - name: Add Kubernetes repository
    blockinfile:
      path: /etc/yum.repos.d/kubernetes.repo
      state: present
      create: yes
      block: |
        [kubernetes]
        name=Kubernetes
        baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
        enabled=1
        gpgcheck=1
        gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
        exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni

- name: Download and Configure Containerd
  hosts: all
  gather_facts: no
  tasks:
    - name: Download containerd
      get_url:
        url: https://github.com/containerd/containerd/releases/download/v2.0.0/containerd-2.0.0-linux-amd64.tar.gz
        dest: /tmp/containerd-2.0.0-linux-amd64.tar.gz

    - name: Extract containerd
      unarchive:
        src: /tmp/containerd-2.0.0-linux-amd64.tar.gz
        dest: /usr/local
        remote_src: yes

    - name: Create systemd directory
      file:
        path: /usr/local/lib/systemd/system
        state: directory
        mode: '0755'

    - name: Download containerd.service
      get_url:
        url: https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
        dest: /usr/local/lib/systemd/system/containerd.service
        mode: '0644'

    - name: Copy runc to /usr/local/sbin
      copy:
        src: /usr/bin/runc
        dest: /usr/local/sbin/runc
        mode: '0755' 

    - name: Reload systemd daemon
      systemd:
        daemon_reload: yes

    - name: Enable and start containerd service
      systemd:
        name: containerd
        state: started
        enabled: yes

    - name: Create CNI bin directory
      file:
        path: /opt/cni/bin
        state: directory
        mode: '0755'

    - name: Download CNI plugins
      get_url:
        url: https://github.com/containernetworking/plugins/releases/download/v1.6.1/cni-plugins-linux-amd64-v1.6.1.tgz
        dest: /tmp/cni-plugins-linux-amd64-v1.6.1.tgz

    - name: Extract CNI plugins
      unarchive:
        src: /tmp/cni-plugins-linux-amd64-v1.6.1.tgz
        dest: /opt/cni/bin
        remote_src: yes 

    - name: Create containerd config directory
      file:
        path: /etc/containerd
        state: directory
        mode: '0755'

    - name: Copy containerd config file from controller
      copy:
        src: kube_config.toml  
        dest: /etc/containerd/config.toml
        mode: '0644'
      
    - name: Restart containerd
      ansible.builtin.systemd_service:
        name: containerd.service
        state: restarted

- name: Install Kubelet/Kubectl/Kubeadm
  hosts: all
  gather_facts: no
  tasks:
    - name: Download libnetfilter_cttimeout
      get_url:
        url: https://mirror.stream.centos.org/9-stream/AppStream/x86_64/os/Packages/libnetfilter_cttimeout-1.0.0-19.el9.x86_64.rpm
        dest: /tmp/libnetfilter_cttimeout-1.0.0-19.el9.x86_64.rpm

    - name: Download libnetfilter_cthelper
      get_url:
        url: https://mirror.stream.centos.org/9-stream/AppStream/x86_64/os/Packages/libnetfilter_cthelper-1.0.0-22.el9.x86_64.rpm
        dest: /tmp/libnetfilter_cthelper-1.0.0-22.el9.x86_64.rpm

    - name: Download libnetfilter_queue
      get_url:
        url: https://mirror.stream.centos.org/9-stream/AppStream/x86_64/os/Packages/libnetfilter_queue-1.0.5-1.el9.x86_64.rpm
        dest: /tmp/libnetfilter_queue-1.0.5-1.el9.x86_64.rpm

    - name: Download conntrack-tools
      get_url:
        url: https://mirror.stream.centos.org/9-stream/AppStream/x86_64/os/Packages/conntrack-tools-1.4.7-4.el9.x86_64.rpm
        dest: /tmp/conntrack-tools-1.4.7-4.el9.x86_64.rpm

    - name: Import CentOS 9 Stream GPG key
      rpm_key:
        key: https://www.centos.org/keys/RPM-GPG-KEY-CentOS-Official
        state: present

    - name: Install conntrack-tools
      ansible.builtin.dnf:
        name: 
          - /tmp/libnetfilter_cttimeout-1.0.0-19.el9.x86_64.rpm
          - /tmp/libnetfilter_cthelper-1.0.0-22.el9.x86_64.rpm
          - /tmp/libnetfilter_queue-1.0.5-1.el9.x86_64.rpm
          - /tmp/conntrack-tools-1.4.7-4.el9.x86_64.rpm
        state: present
    
    - name: Install kubeadm, kubelet and kubectl
      dnf:
        name:
          - kubeadm
          - kubectl
          - kubelet
        state: present
        disable_excludes: kubernetes

    - name: Enable and start kubelet service
      service:
        name: kubelet
        state: started
        enabled: yes

- name: Initialize kubeadm
  hosts: Manager
  gather_facts: yes
  vars:
    pod_cidr: 10.10.0.0/16

  tasks:
    - name: Check if Kubernetes is already initialized
      stat:
        path: /etc/kubernetes/admin.conf
      register: kubeadm_config

    - name: Initialize kubeadm
      shell: |
        kubeadm init \
          --apiserver-advertise-address={{ ansible_host }} \
          --apiserver-cert-extra-sans={{ ansible_host }} \
          --pod-network-cidr={{ pod_cidr }} \
          --cri-socket=unix:///var/run/containerd/containerd.sock \
          --control-plane-endpoint={{ ansible_host }}
      when: not kubeadm_config.stat.exists
      args:
        executable: /bin/bash

    - name: Set up kubeconfig for root
      block:
        - name: Create .kube directory
          ansible.builtin.file:
            path: /root/.kube
            state: directory
            mode: 0755

        - name: Copy admin.conf to kubeconfig
          ansible.builtin.copy:
            src: /etc/kubernetes/admin.conf
            dest: /root/.kube/config
            remote_src: yes
            mode: 0775

        - name: Set ownership of kubeconfig
          ansible.builtin.file:
            path: /root/.kube/config
            owner: root
            group: root
            mode: '0600'
      when: not kubeadm_config.stat.exists

- name: Install CNI (calico)
  hosts: Manager
  gather_facts: false
  tasks:
    - name: Pull tigera calico operator
      get_url:
        url: https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml
        dest: /root/tigera_operator.yaml

    - name: Apply tigera calico operator
      shell: kubectl create -f /root/tigera_operator.yaml
      args:
        executable: /bin/bash
      register: result7
      changed_when:
        - result7.rc == 0
    
    - name: Pull custom-resources for calico
      get_url:
        url: https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/custom-resources.yaml
        dest: /root/calico_custom-resource.yaml

    - name: Replace CIDR in /root/calico_custom-resource.yaml
      replace:
        path: /root/calico_custom-resource.yaml
        regexp: 'cidr: 192\.168\.0\.0/16'
        replace: 'cidr: 10.10.0.0/16'

    - name: Apply tigera calico operator
      shell: kubectl create -f /root/calico_custom-resource.yaml
      args:
        executable: /bin/bash
      register: result8
      changed_when:
        - result8.rc == 0

    - name: Wait for Calico Pods to Initialize
      pause:
        minutes: 2

    - name: Remove taint from control plane nodes
      shell: kubectl taint nodes --all node-role.kubernetes.io/control-plane-
      args:
        executable: /bin/bash
      register: result4
      changed_when:
        - result4.rc == 0

- name: Generate and Distribute Kubeadm Join Token
  hosts: Manager
  tasks:
    - name: Get kubeadm join token
      shell: kubeadm token create --print-join-command
      args:
        executable: /bin/bash
      register: join_token
      changed_when: join_token.rc == 0

    - name: Save token to local file on Ansible control node
      delegate_to: localhost
      copy:
        content: "{{ join_token.stdout }}"
        dest: "/tmp/join_token.txt"

- name: Join worker Nodes
  hosts: Workers
  gather_facts: yes
  tasks:
    - name: Copy token file from Ansible control node to workers
      copy:
        src: "/tmp/join_token.txt"
        dest: "/tmp/join_token.txt"
        
    - name: Get token from file
      command: cat /tmp/join_token.txt
      register: token_from_file

    - name: Register nodes
      shell: "{{ token_from_file.stdout }}"
      args:
        executable: /bin/bash
      register: result5
      changed_when:
        - result5.rc == 0

```
#More information
To get access to these scripts and more, please feel free to visit my github page:
[K8s Install](https://github.com/TNiconic/Personal_Playbooks/tree/main/kube_deploy)

Again, thank so much for taking the time to read and have a great day!