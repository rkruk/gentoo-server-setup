***
<p align="center">![Gentoo](https://github.com/rkruk/gentoo-server-setup/blob/master/larry.png)
<br>
#### <p align="center">**Gentoo Server Setup** 

<p align="center">Set Up your own Gentoo Web Server <br> or Cloud VPS

------------------------------------------------------------------------

----------
<br>
<p align="center">***
<p align="center">**Introduction**
<br>
<p align="center">First of all let me state:
<br><br>
**-I'm not insane**.  ¯\\_(ツ)_/¯
<br><br>
**-It is really worth the effort**.
<br><br>
-Contrary to popular beliefs Gentoo is not a time consuming fringe distro.
<br>
<br>
I have a few Gentoo based servers with a long uptime (not like it is a bad thing..) - and I bet there are working a way better than any Ubuntu, CentOS,etc.. servers I've used in the past.<br><br> *I know - this is a* **bold** *statement to say things like that. But let me explain*:
<p align="left">1. Continous uptime without the need of reboot to switch to the new version of the kernel, services,etc..
<p align="left">2. System is fully customisable, every single part can be adjusted as you like it, every daemon, service - you decide what you want - not the distro maintainers with tedious dependencies forcing you to use THAT version of software. Example: NTP daemon is quite resource heavy for a small daemon - I chenged it openntpd. Same thing with log damemons, web servers,.. 
<p align="left">3. Low resource footprint (for real - it is lower than anything else I have used in the past).
<br>
<br>
<p align="center">***
<p align="center">**Hosting**
<p align="left">There are a few VPS providers out there allowing you to set up your own Gentoo VPS directly (Shout out to <a href="https://linode.com">Linode</a>) or to install your own ISO (Awesome people at<a href="https://wiki.gandi.net/en/hosting/create-server/private-image"> Gandi.net</a>).
<p align="left">-I'm sure there is more providers I'm not aware of offering custom ISO install allowing you to install Gentoo on their infrastructure.
<p align="left">-If you are lucky to have your own server (not the cloud thingy) I'm sure you can use this howto without any problems or changes.
<p align="left">*I'm going to use Linode's VPS as an installation example here. Adapt it to your own needs as you like.*
<p align="left">I won't cover here installation process - Linode will roll it out for you automatically with the help of their installation scripts - just do some magic with with help of your mouse and keyboard. It is straightforward process and it is extremely easy to do that. Otherwise install it on your own with the use of one of the best documentations written ever at the <a href="https://www.gentoo.org/">gentoo.org</a>.
<br>
<br>
<p align="center">*** 
<p align="center">**First Steps**<br>
<p align="left">*Lets assume that you already installed your Gentoo based VPS and you know how to connect through SSH onto it. Your server should be up and running ofcourse.*
<br>
<p align="left">Lets start and log in to our new server through the SSH. Enter the following into your terminal window or application. Be sure to replace the example IP address with your Linode’s IP address (Linode users: you can find it in the --> 'Linodes' tab --> 'Remote Access' tab). Change example address into your VPS address. As a example here I'll use IP: 172.16.254.1 (IPv4 address taken from wikipedia article about IP :/):
```bash
ssh root@172.16.254.1
```
Yes, first login is as a root user. We'll change it later during the configuration of our new server.<br>
First things first let's synchronize server repositories:
```bash
emerge --sync
```
I would recommend update the whole thing (at Gentoo it is better to know about any problems at the begining):
```bash
emerge -uavDN @world 
```
I hope you came here with some knowledge already, and there is no need to explain what this command does.
<br>
After that let's set up a Hostname and fully qualified domain name (FQDN):
Enter the following commands to set the hostname, replacing hostname with the hostname of your choice:
```bash
echo "HOSTNAME=\"hostname\"" > /etc/conf.d/hostname
```
and then:
```bash
/etc/init.d/hostname restart
```
Next lets update the `/etc/hosts` file. This file creates static associations between IP addresses and hostnames, with higher priority than DNS. In the example below, 172.16.254.1 is our public IP address, hostname is our local hostname, and hostname.example.com is our FQDN. Your `/etc/hosts` file should look like that:
```bash
127.0.0.1 localhost.localdomain localhost
203.0.113.10 hostname.example.com hostname
```
<br>
`hostname.example.com` is our FQDN here. The value you assign as your system’s FQDN should have an “A” record in DNS pointing to your Linode’s IPv4 address.
<br><br>
As you can see here the configuration files are self explanatory. When you changed all informations in the `hosts` file do: CTRL + X (save & exit) and ENTER to confirm.
<br><br>
Setting up the Timezone:<br>
View the list of all available zone files:
```bash
ls /usr/share/zoneinfo
```
And manually symlink a zone file in /usr/share/zoneinfo to /etc/localtime:
```bash
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
```
As I'm based in the UK I set it up in this example for the UK based zone.
<br>
Confirm the new settings:
```bash
date
```
<br>
Add a Limited User Account for day-to-day use (in this example I choose a 'Larry' username):<br>
```bash
useradd -m -G users,wheel -s /bin/bash larry
```
and set up a password for a new user:
```bash
passwd
```
<br>
With the new user set you can now log out from your VPS. There is no need to use a root account to log into your server (it is considered as a extremely dangerous though - we'll block that option later).
```bash
exit
```
Log back in as your new user(here it will be: 'Larry'). Remember to change IP address to your VPS IP address:
```bash
ssh larry@172.16.254.1
```
Now you can administer your server from your new user account instead of root. If you want to become a `root` just use the:
```bash
su -
```
command, make neccessary changes and `exit` to become a normal user again. 
<br>
<br>
<br>
<p align="center">***
<p align="center">**Harden SSH Access:**
<br>
<p align="left">By default, password authentication is used to connect to your VPS via SSH. A cryptographic key pair is more secure because a private key takes the place of a password, which is generally much more difficult to brute-force.
<br>
<br>
<p align="center">Create an Authentication Key-pair:
<br>
<p align="center">**!!! This is done on your local computer, not on your VPS !!!** 
<p align="left">We are going to create a 4096-bit RSA key pair here. During creation of the key, you will be given the option to encrypt the private key with a passphrase. This means that key cannot be used without entering the passphrase. I suggest to use the key pair with a passphrase.
```bash
ssh-keygen -b 4096
```
Press `Enter` to use the default names id_rsa and id_rsa.pub in /home/your_username/.ssh - before entering your passphrase.<br>
Next let's upload the public key to your server. Replace `larry` user with the name of the user you created on the server, and 172.16.254.1 with your VPS IP address.<br>
From your local computer:
```bash
ssh-copy-id larry@172.16.254.1
```
<p align="center">**Or if you prefer `scp` command:**<br><br>
On the VPS create `.ssh` directory and change permissions for it:
```bash
mkdir -p ~/.ssh && sudo chmod -R 700 ~/.ssh/ 
```
From your local computer:
```bash
scp ~/.ssh/id_rsa.pub example_user@172.16.254.1:~/.ssh/authorized_keys
```
###To Do:
<br>
server configuration, security, software, services,etc...
<br><br>
<p align="center">*** 
<p align="center">**NGINX Web Server**<br>
<p align="left">While I am not aware of any Gentoo specific benchmarks that have been run on the various flavors of web servers, it is typically logical to assume that the more feature-enabled the version of web server you use, the more resources it would use.<br>
Though Nginx is better in the raw number of requests per second it can serve than Apache or Lighttpd. At higher levels of concurrency, it can handle fewer requests per second, but still can serve double what Lighttpd does (which is already doing nearly 4x what Apache was ever able to do).<br>
<p align="center">![Gentoo](https://github.com/rkruk/gentoo-server-setup/blob/master/Webserver_requests_graph.jpg)<br>
<p align="left">Though Apache supports a larger toolbox of things it can do immediately after install, but it can be something of a memory whore with the more modules enabled. Contrary Nginx does not eat as much memory compared to Apache with enabled modules.<br>
