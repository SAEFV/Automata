# Project Automata: Puppet, Kubernetes & Docker Setup

## Introduction
This document details my project to explore and document the setup of Puppet, Kubernetes, and Docker. I will use VMware to create virtual machines, with one acting as the Puppet master and two as nodes. My operating system of choice is Ubuntu Server.
The goal of this project is to learn how these technologies interact, automate system configurations with Puppet, manage containers with Docker, and orchestrate them using Kubernetes.

---
## VM Setup
Puppet master
- user & pass: odin
- hostname: asgard
-  network adapter 1: NAT
-  network adapter 2: LAN (192.168.0.10)
-  Install Type: Default Ubuntu Server
    - status: done

**Node 1**
- user & pass: loki
- hostname: jotunheim
- network: NAT
- network adapter 2: LAN (192.168.0.11)
- Install Type: Minimized Ubuntu Server
- Featured sever snaps: microk8s & docker
    - status:  done

**Node 2**
- user & pass: freya
- hostname: vanaheim
- network: NAT
- network adapter 2: LAN (192.168.0.12)
- Install Type: Minimized Ubuntu Server
- Featured server snaps: microk8s & docker
    - status: done

### conclusion
Three virtual machines have been created using VMware, all configured with NAT networking. The Puppet master (asgard) will manage two agents (jotunheim and vanaheim). Both agent nodes have installed MicroK8s and Docker via the Ubuntu Server installer's Featured Server Snaps.
Needed to add a LAN adapter to the servers because NAT does not let the servers communicate. 
**Note**: Minimized Ubuntu is *very* stripped down — it doesn't include tools like `nano`, `vim`, or even `ping`, and lacks shell autocompletion. It's cool discovering all the missing pieces.

---
## Installing and Configuring Puppet
Set Up Hostname Resolution Ensure that each machine can resolve the other's hostname by editing the /etc/hosts file on all nodes.
Ping test from (asgard) to nodes.
```bash
ping jotunheim #Success
ping vanaheim #Success
```

#### Add Puppet Repository:
Download and install the Puppet release package:

```bash
wget https://apt.puppet.com/puppet8-release-jammy.deb
sudo dpkg -i puppet8-release-jammy.deb
sudo apt update
```

#### Install Puppet Server:
Install the Puppet Server package:
```bash
sudo apt install puppetserver -y
```
### Puppet Master Configuration
After installing Puppet on the master (Asgard), we configure Puppet by editing the /etc/puppetlabs/puppet/puppet.conf file. By default, this file only contains comments.

We add the following lines under the [main] section:
```bash
[main]
certname = asgard
server = asgard
environment = production
runinterval = 1h
```
Explanation:
- certname = asgard: Sets the certificate name to match the hostname of the Puppet master.
- server = asgard: Tells Puppet agents to connect to this hostname.
- environment = production: Specifies the default environment for applying manifests.
- runinterval = 1h: Sets how often the agent checks in (default is 30 minutes, adjusted here for a slower lab cycle).

This configuration helps ensure consistency across your Puppet environment and makes your setup more predictable.

#### Add Puppet to your shell's PATH:
```bash     
echo 'export PATH=$PATH:/opt/puppetlabs/bin' >> ~/.bashrc
source ~/.bashrc
```
Installing Puppet Agent on Nodes (Jotunheim & Vanaheim)
To ensure the Puppet agent version matches the master (Asgard), we install it from the official Puppet APT repository.
Steps:
1. Add the Puppet 8 APT repository (for Ubuntu 22.04):
```bash
    wget https://apt.puppet.com/puppet8-release-jammy.deb
    sudo dpkg -i puppet8-release-jammy.deb
    sudo apt update
```
2. Install the Puppet agent:
```bash
sudo apt install puppet-agent
```
3. Add Puppet to your shell's PATH (optional, makes puppet command globally available):
```bash
echo 'export PATH=$PATH:/opt/puppetlabs/bin' >> ~/.bashrc
source ~/.bashrc
```
4. Verify installation:
```bash
puppet --version
```
### Puppet Agent Configuration
Once Puppet is installed on the agent machines (Jotunheim and Vanaheim), we configure them to connect to the Puppet master (Asgard).
Edit the config file on each node:
sudo nano /etc/puppetlabs/puppet/puppet.conf
Then add the following under the [main] section:
```yaml
[main]
certname = jotunheim       # or vanaheim, depending on the node
server = asgard
environment = production
runinterval = 1h
Make sure certname matches the hostname of the node you're editing.
```
Explanation:
- certname: Identifies the node to the Puppet master. Must match the VMs hostname.
- server = asgard: Tells the agent which server to request configurations from.
- environment = production: Keeps all nodes in the same working environment.
- runinterval = 1h: Sets the frequency of communication between agent and master.

After saving the configuration, you can manually initiate a connection from the agent to the master with:
```bash
sudo puppet agent --test
```
**!Note**: this will fail see fix below

#### Making Puppet Globally Accessible (fix)
By default, Puppet is installed to /opt/puppetlabs/bin, which is not included in the system-wide PATH. This means running puppet as sudo or from scripts may not work unless you provide the full path. To fix this, we create a symbolic link (symlink) to the puppet binary in /usr/local/bin, which makes it accessible globally:
```bash
sudo ln -s /opt/puppetlabs/bin/puppet /usr/local/bin/puppet
```
##### Why this is safe:
- It does not modify the actual binary — it simply adds a shortcut.
- /usr/local/bin is intended for user-added tools and safe customizations.
- Puppet updates will not overwrite this symlink.

After this step, you can run puppet normally from any shell or script:
```bash
sudo puppet agent –test
```

You can also create similar symlinks for other useful Puppet tools like:
```bash
sudo ln -s /opt/puppetlabs/bin/facter /usr/local/bin/facter
sudo ln -s /opt/puppetlabs/bin/hiera /usr/local/bin/hiera
```
**Note**: new error connection to asgard failed TCP connection to asgard:80140 refused. I accidentally installed an older version of the Puppet agent on the master using an outdated wget link. I reinstalled Puppet 8 on all VMs and updated the document accordingly. trying fix and updating the document. I needed puppet8 I will have to reinstall puppet-agent on the nods as well dam it. Reinstalled puppet on all VMs to the correct version edited the doc. Connection

---
### Sign the Puppet Agent Certificate on Asgard
Once the Puppet agent (e.g., jotunheim) sends a certificate signing request (CSR), the Puppet master (asgard) must approve it. Follow these steps:
1. Check for pending certificate requests on Asgard:
```bash
       sudo puppetserver ca list --all
```
2. Sign the agent’s certificate: Replace jotunheim with the actual certname if needed.
```bash
       sudo puppetserver ca sign --certname jotunheim
```
3. Verify the agent is trusted by rerunning the agent command on jotunheim:
```bash
       sudo puppet agent --test
```
A successful connection will show messages like "Applied catalog" and "Notice: Applied X resources".
#### Agent Test and Catalog Application on Jotunheim
After signing the certificate on the Puppet Server (Asgard), test the Puppet agent on the node (Jotunheim):
sudo puppet agent --test
**Example output**:
```bash
Info: Creating a new SSL certificate request for jotunheim
Info: Downloaded certificate for jotunheim from https://asgard:8140/puppet-ca/v1
Info: Using environment 'production'
Info: Retrieving pluginfacts
Info: Retrieving plugin
Notice: Requesting catalog from asgard:8140 (192.168.0.10)
Notice: Catalog compiled by asgard
Info: Caching catalog for jotunheim
Info: Applying configuration version '1743801244'
Notice: Applied catalog in 0.02 seconds
```
This confirms that:
- The agent-server communication is working
- The Puppet catalog is applied correctly
- SSL certificate was issued and accepted


Next Steps
- Document Node Setup: Finish detailing how Jotunheim and Vanaheim are configured as Puppet agents.
- Create and Test Puppet Manifests:
    - Write your first Puppet .pp file to manage packages, services, or files.
    - Apply it manually on Jotunheim or Vanaheim.
- Automate Manifest Application:
    - Configure manifests to be applied automatically based on environment or node.
- Begin Docker Integration (optional):
    - Experiment with using Puppet to manage Docker or prepare for Kubernetes setup.
- Prepare Kubernetes Nodes:
    - Decide which VM(s) will run kubeadm and act as control plane and workers.
    - Document network, hostname, and roles like you did for Puppet.
- Test and Debug:
    - Note any errors or lessons learned, especially when Puppet and Kubernetes start interacting.

**Note**: generated "next steps" with our savior ChatGPT im calling it quits for this session. I got puppet configured and ca singed by (asgard) fix some mistakes in the document.

**Note**: just got done converting the doc form .odt to markdown and fixing spelling.

---

