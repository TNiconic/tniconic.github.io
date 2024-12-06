---

layout: post
title: Validating Vmware Security
date: 2023-08-09 12:00:00 -500
categories: [vmware,vcenter,stig,vulnerabiiltycheck]
tags: [vmware,vcenter,stig,vulnerabiiltycheck]

---

# VMWare Vuln Check
Hello! And welcome back to my blog, today we're going to be exploring something a little different; which is the topic of cybersecurity governance and cybersecurtiy vulnerability assessment. But what do these terms really mean? In short, we are checking the validity and health of a system in the scope of a risk assessment; using pre-defined "rules/criteria" that the United States government utilizes to determine if a system meets their operating standard. Even more blunty; the government publishes guides for specific systems that are meant to be used to determine the "health" of a particular system. This is great news for us! This means that individuals have already done the hard work determining security controls and best practices that we can follow to secure our systems.... only one problem.

These checks can be long, cumbersome and often confusing for even a skilled CyberSecurity practitioner. Which leads me to the purpose of this blog and the mini-project I've created. But before we get too in the weeds, let's go over some concepts.

![shellscript](https://media.giphy.com/media/PjDzu5ZAsSgnrLhms9/giphy.gif)

# Overview of Stig Viewer
Before we can really go over the project, we must first talk about Stig Viewer, our focus technology and how we can use this to our advantage.

![collector](/assets/vmwarepost/stig.png)

First, what is a stig?
STIG stands for Security Technical Implemntation Guide. As previously mentiond, it is a set of guielines and best practices developed by DISA and used by the United States Government. STIGs are designed to provide standardization and enhance the overall security posture of IT systems by creating a common baseline and have technology adhere to it, creating a series of uniform systems. This really boils down to sets of instructions on how to secure systems based off their area of technology. The main technology we will be focusing on in this blog post is the vSphere (VMware) STIG. This STIG and the necessary tool to view this and many other stigs will be listed below.

[StigViewer](https://public.cyber.mil/stigs/downloads/) Type viewer in the searchbar and select your appropriate version.
[VMwareStig](https://public.cyber.mil/stigs/downloads/) Type vsphere in the searchbar and it will be the newest version.

# Why VMWare?
The reason for this guide is the prominence of VMware and its suite of products in the wild and DoD. The vSphere (VMware cloud hypervisor manager) STIG encompasses 10 different checklists that total over 300 different checks for security compliance. This obviously can take quite some time per site, and if they have multiple VMware resources (think multiple ESXI host enclaves) it can be unrealistic to constantly be checking and validating these nodes manually. This brings us to the reason for this post; I have scripted and automated the process and divided it into 3 sections, as each have their own pre-requisites to work. However, keep in mind that I wrote these scripts to work on all DoD systems, and operability was my main focus. With that said let's get started!

# Part 1: VSphere vCenter Appliance Checks
The bulk of the checks you will be running will be against the vCenter Appliance (think management utility for vSphere hypervisors) and VAMI (web-based management interface). Listed below are the automated checklists for this script:

1. VMware vSphere 7.0 VAMI Security Technical Implementation Guide || 28 Checks
2. VMware vSphere 7.0 vCenter Appliance EAM Security Technical Implementation Guide || 33 Checks
3. VMware vSphere 7.0 vCenter Appliance Lookup Service Security Technical Implementation Guide || 31 Checks
4. VMware vSphere 7.0 vCenter Appliance Perfcharts Security Technical Implementation Guide || 34 Checks
5. VMware vSphere 7.0 vCenter Appliance Photon OS Security Technical Implementation Guide || 113 Checks
6. VMware vSphere 7.0 vCenter Appliance PostgreSQL Security Technical Implementation Guide || 20 Checks
7. VMware vSphere 7.0 vCenter Appliance RhttpProxy Security Technical Implementation Guide || 8 Checks
8. VMware vSphere 7.0 vCenter Appliance STS Security Technical Implementation Guide || 31 Checks
9. VMware vSphere 7.0 vCenter Appliance UI Security Technical Implementation Guide || 33 Checks

The only pre-requisites to get his script working is for you to have the ability to SSH into the vSphere client (running as a VM on an ESXI host) as root. Once connected, you will just need to create a text/.sh file and copy the contents of the vSphere.sh script into your newly created file; save and exit and then give it execute permissions. Then run the script and you will be presented with the results of your compliance.
Link to github repo: [vSphere.sh_gh](https://github.com/TNiconic/CCRI/blob/main/VMware/vSphere.sh)

![vsphere.sh](https://media.giphy.com/media/EbTCLtFlfF3MekXAHV/giphy.gif)

# Part 2: ESXi Host Checks
The next script will check individual ESXi hosts for their compliance. You can have multiple ESXi hosts nested within 1 vCenter distribution, however for this script you will need to ssh into each inidiviual ESXi host and run the script. You can run this against as many ESXi hosts as necessary. Listed below are the automated checklists for this script:
1. VMware vSphere 7.0 ESXi Security Technical Implementation Guide || 75 Checks

The pre-requisites to get this script working are as follows. You must first have PowerCLI installed; this is a powershell extension that adds VMware functionality. From and administrative Powershell console:

```Powershell
Find-Module -Name VMware.PowerCLI
Install-Module -Name VMware.PowerCLI -Scope CurrentUser
Get-Command -Module *VMWare*
```

You will then need to enable unsigned powershell scripting on this machine. This let's you run scripts (such as this one) from "untrusted sources". It is highly recommended to revert this change after running this script. From and administrative Powershell console:

```Powershell
set-executionpolicy remotesigned
#Press Y to only apply to the local user
```

Some of the functionality of the script requires they have PuTTY installed. For those that don't know; PuTTY is a application that allows for remote management on Windows and other OS's and is widely used. If you don't have putty installed then the script will function but certain checks will error out.

You will need to enable SSH on the target ESXi host via the Web GUI. Below is an example of doing just that:

![ESXiSSH](/assets/vmwarepost/esxi.png)

Connect to the ESXi host using the below PowerCLI command:

```Powershell
Connect-ViServer {hostname/IPaddress}
```

Save the script listedbelow and run it. It's best advised to navigate to the directory your script is placed before hand. Below is a GIF demonstration (*You won't see me inputting credentials but you do get prompted*):

![esxi.ps1](https://media.giphy.com/media/pFQKyCXNxtKF4DGTqq/giphy.gif)


Link to github repo: [esxi7.ps1_gh](https://github.com/TNiconic/CCRI/blob/main/VMware/esxi7.ps1)

# Part 3: Virtual Machine Checks
The last checks we will be looking at are those that involve checking individual virtual machines. This script works most effectively if you can ssh into an ESXi/vCenter host that has nested virtual machines, that way you can check multiple virtual machines at one time. Listed below are the automated checklists for this script:
1. VMware vSphere 7.0 Virtual Machine Security Technical Implementation Guide

If you've followed the steps in Part 2, then you will not have any further pre-requisites to complete. Just make sure that you connect to the appropriate ESXi host / vCenter host that has this virtual machine nested and that you've already downloaded the script provided below.

Once connected run the script; you will then be prompted to enter how many Virtual Machines you want to evaluate. For each Virtaul Machine, you will be prompted to enter the hostname which will then run the script agains. Below is a GIG demonstration:

![virtualmachines.ps1](https://media.giphy.com/media/s58syCtOJPC2inQZDK/giphy.gif)

Link to github repo: [VirtualMachines.ps1_gh](https://github.com/TNiconic/CCRI/blob/main/VMware/VirtualMachines.ps1)


# Conclusion
There you go, after running the scripts you will see the output of every STIG check and whether they are considered "Not a Finding" (meets security standard) or "Open" (does *NOT* meet security standard). If the output is open it will also show you the output of the command that you can cross reference with the STIG check. I hope this proves to be very useful, and if you have any quesions or things you would like to add please feel free to reach out to me via email or other contact information. Thanks for taking the time to read this and I hope you have a good day!