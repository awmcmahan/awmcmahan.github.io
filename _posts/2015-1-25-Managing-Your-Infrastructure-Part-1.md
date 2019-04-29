---
layout: post
title: 'Managing Your Infrastructure Part 1: Setup Foreman'
date: '2016-1-24'
categories: foreman puppet
---

# Why Foreman?
We will be going with Foreman because it provides a web interface and a central location for the agents to check in. It also provides a way for us to manage our nodes and view their puppet runs. In this guide I will be using an Ubuntu 14.04 LTS server. **This is a fresh install with all available updates.**

# Setup Foreman
We will be using the Foreman Installer, which is program that runs a puppet module to setup our Foreman server. It should be noted that this **will make make changes to databases and web server configuration** so make sure this is backed up or you have nothing to lose. It will setup and install Foreman, a puppet master with passenger and the Smart Proxy by default. At the time of writing this the current version is 1.10 so you might want to look to see if there are any newer versions before continuing as upgrading, in my experience, is pretty unreliable.

**Setup Fully Qualified Domain**<br> Ensure that `ping $(hostname -f)` shows the real IP address and not 127.0.0.1. If you already have this setup feel free to skip this step.

Edit your hosts file so that it is similar to this:

```bash
$ sudo nano /etc/hosts
    127.0.0.1       localhost
    127.0.1.1      foreman.domain.com foreman

    # The following lines are desirable for IPv6 capable hosts
    ::1     localhost ip6-localhost ip6-loopback
    ff02::1 ip6-allnodes
    ff02::2 ip6-allrouters
```

And then your hostname file:

```bash
$ sudo nano /etc/hostname
foreman.domain.com
```

**Enable the repository:**

```bash
$ sudo apt-get -y install ca-certificates
$ wget https://apt.puppetlabs.com/puppetlabs-release-trusty.deb
$ dpkg -i puppetlabs-release-trusty.deb
```

**Enabling the Foreman repo:**

```bash
$ echo "deb http://deb.theforeman.org/ trusty 1.10" > /etc/apt/sources.list.d/foreman.list
$ echo "deb http://deb.theforeman.org/ plugins 1.10" >> /etc/apt/sources.list.d/foreman.list
$ wget -q http://deb.theforeman.org/pubkey.gpg -O- | apt-key add -
```

**Run the installer:**<br>The installation is non-interactive by default but it does come with options to modify its behavior. Be sure to check out the options listed on their [website](http://theforeman.org/manuals/1.10/index.html#3.2ForemanInstaller "Foreman Install Options") but for our setup the default behavior should be more than enough.

```bash
$ sudo apt-get update
$ sudo apt-get -y install foreman-installer
$ sudo foreman-installer
```

After it completes, you should be given information about where to find the servers. Be sure to save this information for the future. If it was successful your output should be similar to this:

```bash
* Foreman is running at https://foreman.domain.com
      Initial credentials are admin / 3ekw5xtyXCoXxS29
  * Foreman Proxy is running at https://foreman.domain.com:8443
  * Puppetmaster is running at port 8140
  The full log is at /var/log/foreman-installer/foreman-installer.log
```

From here you can access your Foreman web interface through your web browser by entering the URL listed in the output.

 ![]({{ site.baseurl }}/content/images/foreman-Login.png)

--------------------------------------------------------------------------------

# Configure Foreman
 Now that you have your Foreman server up and running, let's make some changes. We will create a new administrator account, enable puppet and Foreman on boot and install our first puppet module.  

**Create New Administrator User** <br> In the upper right hand side, hover over **Administer** and click **Users**.

![]({{ site.baseurl }}/content/images/foreman-users.png)

Create a new user enter the desired information and then click the **Roles** tab and check the **Administrator** box.

![]({{ site.baseurl }}/content/images/foreman_administer.png)

Now you have a new administrator. Let's log out of the root account and login as our new administrator.

## Have Foreman and Puppet start at Boot
Enabling Foreman and puppet is quite simple. We will need to edit their config files and tell them the agent to run.

**Foreman:**

```bash
$ sudo nano /etc/default/foreman
  # Start foreman on boot?
  START=yes
```

**Puppet Agent:**

```bash
$ sudo nano /etc/default/puppet
    # Enable puppet agent service?
    # Setting this to "yes" allows the puppet agent service to run.
    # Setting this to "no" keeps the puppet agent service from running.
    START=yes
```

## Enable diffs
Diffs enable you to view what has been modified by puppet in a file. It will show you what was removed and what replaced the content.

```bash
$ sudo nano /etc/puppet/puppet.conf
    show_diff     = true
```

**Install first Puppet module:**

```bash
$ sudo puppet module install -i /etc/puppet/environments/production/modules saz/ntp
```

--------------------------------------------------------------------------------

# Setup the agent
The puppet service will run every 30 minutes by default but it can be configured in the puppet.conf file by changing the runinterval to the number of minutes desired.  

**Manually run our agent:**

```bash
$ sudo puppet agent --test
```

The puppet agent and master communicate over SSL, so before they can talk the agent will generate a new SSL certificate and send it to the master to sign. Once the agent's SSL cert is signed by the master, they can resume communications.  To sign the agent's SSL certificate you will need to login into your Foreman server and select Smart Proxies.

![]({{ site.baseurl }}/content/images/foreman_smart_proxies.png)

You should then see your Foreman server listed as one of the **Smart Proxies**, to the right click **Certificates**.

You should then see your agents SSL cert awaiting to be signed, click **Sign** and you now have your puppet master and agent setup awaiting instructions.