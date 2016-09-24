***
<p align="center">![Gentoo](https://github.com/rkruk/gentoo-server-setup/blob/master/larry.png)
<br>
#### <p align="center">**Gentoo Server Setup** 

<p align="center">Set Up your own Gentoo based Web Server

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
Contrary to popular beliefs Gentoo is not a time consuming left in the past fringe distro. Trough the last 8 years I've used and tested most of existing Linux and *BSD based Distributions - that is a fascinating yet tedious hobby of mine. Don't ask me why.. - it is just my thing. By most I mean, like distrowatch.com from top to bottom (and a few more - non existing any more).  
<br>
<br>
Currently I have a few Gentoo based servers with a quite decent uptime (not like it is a bad thing..) - and I can bet that they are working a way better than any comparable Ubuntu, CentOS, etc.. server I've used in the past.<br><br> *I know - this is a* **bold** *statement to say things like that. But let me explain*:
<p align="left">1. Continous uptime without the need of reboot to switch to the new version of the kernel, services,etc.. (sometimes it is tricky but yet possible (kernel - duh!)).
<p align="left">2. System is fully customisable, every single part can be adjusted as you like it, every daemon, service,.. - you decide what you want - not the distro maintainers with all those tedious dependencies forcing you to use THAT version of THAT software with tons of bloat as a dependecies. You say 'every Linux is customisable' - and I laught :D .<br><br> 
For starters simple example: NTP daemon (`net-misc/ntp`) is quite resource heavy for a small silly daemon - I've changed it to openntpd (NTP server ported from OpenBSD - it use less resources).<br> 
Same thing is with all log damemons, web servers, firewalls, etc.. Yes, you can do things like that on any other distro. But I wonder how much od your's precious time will you waste to do just that? :D
<p align="left">3. Low resource footprint (for real - it is lower than anything else I have ever used and seen in the past - lower than debian - for real). Results are almost similar to LFS if system is set correctly.
<br>
<br>
<p align="center">***
<p align="center">**Hosting**
<p align="left">You can install it on your own hardware if you have it. You can rent a rack somewhere if you have cash to burn. Or you can use VPS. There are a few VPS providers out there allowing you to set up your own Gentoo VPS directly (Shout out to <a href="https://linode.com">Linode</a>) or to install your own ISO (Awesome people at<a href="https://wiki.gandi.net/en/hosting/create-server/private-image"> Gandi.net</a>). I'm aware of the fact that Amazon AWS have Gentoo images, but I haven't used them and I can't say anything about it (folks at [Dowd and Associates](http://www.dowdandassociates.com/) are responsible for those system images).
<p align="left">-I'm sure there is more providers - I'm not aware of - offering custom ISO install allowing you to set your own Gentoo on their infrastructure.
<p align="left">-If you are lucky to have your own server (not the cloud thingy) I'm sure you can use this howto without any problems or changes.
<p align="left">*I'm going to use Linode's VPS as an installation example here. Adapt it to your own needs as you like.*
<p align="left">I won't cover here installation process - Linode will roll it out for you automatically with the help of their installation scripts - just do some magic with with help of your mouse and keyboard. It is straightforward process and it is extremely easy. Otherwise install it on your own with the use of one of the best documentations written ever (just right after *BSD documentation) at the <a href="https://www.gentoo.org/">gentoo.org</a> website.
<br>
<br>
<p align="center">*** 
<p align="center">**First Steps**<br>
<p align="left">*I assume that you already installed your Gentoo based system and you know how to connect through SSH onto it. Your server should be up and running already of course.*
<br>
<p align="left">Lets start and log in to our new server through the SSH. Enter the following into your terminal window or application (putty --> if you are using silly Windows based OS :/ ). Be sure to replace the example IP address with your server IP address (Linode users: you can find it in the &#8594; 'Linodes' tab &#8594; 'Remote Access' tab). As a example here I'll use IP: 123.456.78.9 address:<br>

```bash
ssh root@123.456.78.9
```
<br>
Yes, unfortunately first login is as a root user. We'll change it soon during the initial configuration of our new server.<br>
But first things first let's synchronize server repositories:<br>
```bash
emerge --sync
```
<br>
I would recommend update the whole thing now (at Gentoo it is better to know about any problems at the begining): <br>
```bash
emerge -uavDN system && emerge -uavDN world
```
<br>
I hope you came here with some knowledge already, and there is no need to explain what this command does! If not - stop now and go read some manuals first. Gentoo won't forgive you any lack of knowledge ;)
<br>
When the update and all post-update requests are done... (joke!)<br>
OK. First major update will for sure flag some problems with packages dependecies. Dealing with those errors is sometimes tricky and time consuming. But in Gentoo there is no way you cannot work around it. Gentoo documentation is very detailed and you will love it. Most of the problems are easy to avoid if you follow <a href="https://wiki.gentoo.org/wiki/Handbook:AMD64/Working/Portage">portage documentation</a>.<br> You will be forced to update content of portage configuration files in the `/etc/portage/`like: setting appropriate flags for packages in the `/etc/portage/package.use`, configuring entire system in the `/etc/portage/make.conf `, setting package versions in the `/etc/portage/package.mask`, `/etc/portage/package.unmask` and `/etc/portage/package.accept_keywords`.<br> Just make sure you know what you are doing there!<br>
After that carefully read what `etc-update` and `dispatch-conf` tools are telling you to do and <a href="https://wiki.gentoo.org/wiki/Handbook:X86/Portage/Tools">follow instructions</a>.<br><br>
And finally when the `emerge -uavDN system && emerge -uavDN world` update is done fully and Gentoo accepted your sacrifice :D - you can begin setting up and configure your server for web.<br><br>
Let's start with setting up a Hostname and fully qualified domain name (<a href="http://gentoo-en.vfose.ru/wiki/Fully_Qualified_Domain_Name_Configuration">FQDN</a>) and enter the following commands to set the hostname, replacing hostname with the hostname of your choice:
```bash
echo "HOSTNAME=\"hostname\"" > /etc/conf.d/hostname
```
and then:
```bash
/etc/init.d/hostname restart
```
and/or:
```bash
echo "your hostname" > /etc/hostname
```
Check your hostname:
```bash
hostname -F /etc/hostname
hostname
```

Next lets update the `/etc/hosts` file.<br> 
This file creates static associations between IP addresses and hostnames, with higher priority than DNS!<br> 
In the example below, `123.456.78.9` is our public accessible IP address, `hostname` is our local hostname, and `hostname.example.com` is our FQDN. Your `/etc/hosts` file should look like that:
```bash
127.0.0.1 localhost.localdomain localhost
123.456.78.9 hostname.example.com hostname
```
Don't edit first '127.0.0.1' line, it is for localhost internal server loopback network - but it is highly recommended to add the second line with the right IP address and your Fully Qualified Domain Name.<br>
You are the owner of `example.com` and in the `/etc/hostname` your server has its own name (here it is `hostname` but you can name it as you want it - it doesn't really matter). FQDN does not need to have any relationship to websites hosted on this server. As an example, you might host “example.com” website on your server, but the system’s FQDN might be “xyz.example.com.”<br><br>
It is highly recommended to have registered domain/subdomain on the DNS level for FQDN purposes.
<br><br>
`hostname.example.com` is our FQDN here. The value you assign as your system’s FQDN should have an “A” record in DNS pointing to your's server address.
<br><br>
As you can see here the configuration files are self explanatory.<br>
<br><br>
<!-- Done to this line -->
Setting up the Timezone:<br>
View the list of all available zone files:
```bash
ls /usr/share/zoneinfo
```
And manually symlink a zone file in /usr/share/zoneinfo to /etc/localtime:
```bash
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
```
As I'm based in the UK I'm going to set it up in this example for the UK based zone. 
<br>
Confirm the new settings:
```bash
date
```
<br>
Add a Limited User Account for day-to-day use (in this example I choose a 'larry' username):<br>
```bash
useradd -m -G users,wheel -s /bin/bash larry
```
and set up a password for a new user:
```bash
passwd larry
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

Or add your everyday user to the group 'wheel' and use 'sudo' with all mayor commands (I'm not sure if it is secure wise on server env - need to research that one. I will update it).
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
ssh-keygen -b 8192
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
