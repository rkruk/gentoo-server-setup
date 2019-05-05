<p align="center">
  <img src="https://github.com/rkruk/gentoo-server-setup/blob/master/larry.png">
</p>
<br>
<p align="center"><b>GENTOO SERVER SETUP</b></p>

<p align="center"><b>Set Up your own Gentoo based Web Server</b></p>

------------------------------------------------------------------------

----------
<br>
<p align="center"><b>***</b></p>
<p align="center"><b>Introduction</b></p>
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
<p align="center"><b>***</b></p>
<p align="center"><b>Hosting</b></p>
<p align="left">You can install it on your own hardware if you have it. You can rent a rack somewhere if you have cash to burn. Or you can use VPS. There are a few VPS providers out there allowing you to set up your own Gentoo VPS directly (Shout out to <a href="https://linode.com">Linode</a>) or to install your own ISO (Awesome people at<a href="https://wiki.gandi.net/en/hosting/create-server/private-image"> Gandi.net</a>). I'm aware of the fact that Amazon AWS have Gentoo images, but I haven't used them and I can't say anything about it (folks at [Dowd and Associates](http://www.dowdandassociates.com/) are responsible for those system images).
<p align="left">-I'm sure there is more providers - I'm not aware of - offering custom ISO install allowing you to set your own Gentoo on their infrastructure.
<p align="left">-If you are lucky to have your own server (not the cloud thingy) I'm sure you can use this howto without any problems or changes.
<p align="left">*I'm going to use Linode's VPS as an installation example here. Adapt it to your own needs as you like.*
<p align="left">I won't cover here installation process - Linode will roll it out for you automatically with the help of their installation scripts - just do some magic with with help of your mouse and keyboard. It is straightforward process and it is extremely easy. Otherwise install it on your own with the use of one of the best documentations written ever (just right after *BSD documentation) at the <a href="https://www.gentoo.org/">gentoo.org</a> website.
<br>
<br>
<p align="center"><b>****</b></p>
<p align="center"><b>First Steps</b></p>

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
Hostname can be set also in the:
```bash
echo "your hostname" > /etc/hostname
```
You should check which configuration your server has. just check inside of /etc directory for hostname file or inside /etc/conf.d/. Then pick the right command. Not before!

After that check your hostname:
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
As you can see here all server configuration files are self explanatory :D<br>
<br><br>
Lets continue now with setting up the Timezone for the server.<br>
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
Confirm that the timezone is set correctly:
```bash
date
```
<br>
Add a Limited User Account for day-to-day use with limited rights (in this example I choose a 'larry' username):<br>
```bash
useradd -m -G users,wheel -s /bin/bash larry
```
and set up a password:
```bash
passwd larry
```
Add user to the `wheel` group (`wheel` sudo group so you’ll have administrative privileges):
```bash
usermod -aG wheel larry
```
Check if the `wheel` group is uncommented in the `/etc/sudoers` file:
```bash
nano /etc/sudoers
```
and look for the line:
```bash
%wheel ALL=(ALL) ALL
```
Save all changes.<br>
With the new user set you can now log out from your VPS. There is no need to use a root account to log into your server (it is considered as a extremely stupid and dangerous though - we'll block that option later).
```bash
exit
```
<br>
Log back in with your new user (here it will be: 'Larry'). Remember to use IP address of your server:
```bash
ssh larry@123.456.78.9
```
Now you can administer server from your new user account instead of root. If you really want to become a `root` user just use the:
```bash
su -
```
Make neccessary changes and then `exit` to become a normal user again.<br>
You can also use `sudo` command instead of switching between users:

```bash
sudo example-command
```

Use your everyday user with the 'sudo' priviliges for all mayor administration commands.
<br>
<br>
<br>

<p align="center">***
<p align="center">**Harden SSH Access:**
<br>
<p align="left">By default, password authentication is used to connect to most VPS via SSH (at least at Linode, Nephoscale, AWS, Gandi). And we are going to change that with the use of the SSH keys. A cryptographic key pair is more secure way to log in to your VPS because a private key takes the place of a password, which is generally much more difficult to brute-force. Why?<br>
SSH keys serve as a means of identifying yourself to an SSH server using public-key cryptography and challenge-response authentication. One immediate advantage this method has over traditional password authentication is that you can be authenticated by the server without ever having to send your password over the network. Anyone eavesdropping on your connection will not be able to intercept and crack your password because it is never actually transmitted. Additionally, as I wrote previously using SSH keys for authentication virtually eliminates the risk posed by brute-force password attacks by drastically reducing the chances of the attacker correctly guessing the proper credentials.<br>
I'll use here default RSA based key therefore there is no need to specify `ssh-keygen` with the `-t` option. RSA provides the best compatibility of all algorithms but requires the key size to be larger to provide sufficient security.
Minimum key size is 1024 bits, default is 2048 (see `man ssh-keygen`) and maximum is 16384. 
<br>
<br>
<p align="center">Create an Authentication Key-pair:
<br>
<p align="center">**(This has to be done on your local computer, not on the VPS !!!)**

<p align="left">We are going to create a 8192-bit RSA key pair here. There are voices in the community that we should use bigger, better, stronger keys. But key is only one small part of the security. And right now, a default 2048-bit RSA key, or any greater length (such as the 4096-bit key size of the Github suggestion), is unbreakable with today's technology and known factorization algorithms. Even with very optimistic assumptions on the future improvements of computational power available for a given price (namely, that Moore's law holds at its peak rate for the next three decades), a 3072-bit RSA key won't become breakable within the next 30 years by Mankind as a whole, let alone by an Earth-based organization. Of course, there always remains the possibility of some unforeseen mathematical breakthrough that makes breaking RSA keys a lot easier. Unforeseen breakthroughs are, by definition, unpredictable, so any debate on that subject is by nature highly speculative.<br>If you need more security than default RSA-2048 offers, the way to go would be to switch to elliptical curve cryptography (ed25519)<br>
<!--
     SSH part needs to be revised!
-->
During creation of the key, you will be given the option to encrypt the private key with a passphrase. This means that key cannot be used without entering the passphrase. I suggest to use the key pair with a passphrase.
```bash
ssh-keygen -b 8192
```
Press `Enter` to use the default names id_rsa and id_rsa.pub in /home/your_username/.ssh - before entering your passphrase.<br>
Next let's upload the public key to your server. Replace `larry` user with the name of the user you created on the server, and 172.16.254.1 with your VPS IP address.<br>
From your local computer:
```bash
ssh-copy-id larry@123.456.78.9
```
Or if you prefer `scp` command:<br><br>
On the VPS create `.ssh` directory and change permissions for it:
```bash
mkdir -p ~/.ssh && sudo chmod -R 700 ~/.ssh/ 
```
From your local computer:
```bash
scp ~/.ssh/id_rsa.pub larry@123.456.78.9:~/.ssh/authorized_keys
```
<!-- 
    A bit of security here and there:
-->
For the security reasons I would strongly advise to disallow `root` logins over SSH. All SSH connections will be made by non-root user (larry in this example). Once a limited user account (larry) is connected, administrative privileges are accessible either by using `sudo` or changing to a root shell using `su -` command.<br><br>

Lets edit a `/etc/ssh/sshd_config` file: 
```bash
nano /etc/ssh/sshd_config
```
and change `yes` to `no` in the `PermitRootLogin` line:<br>
```bash
# Authentication:
...
PermitRootLogin no
```
<br><br>
As we are going to use only key based authentication I recommend to disable SSH password authentication. This will require for all users allowed to connect via SSH to use key authentication only. Edit the same file:  
```bash
nano /etc/ssh/sshd_config
```
change the line `PasswordAuthentication yes` to disable clear text passwords:
```bash
PasswordAuthentication no
```
<br>
<b>!!!</b>Though you may want to leave password authentication enabled if you connect to your Linode server from many different computers. This will allow you to authenticate with a password instead of generating and uploading a key-pair for every device.
<br><br>

Continue with another flags for ssh in the `/etc/ssh/sshd_config` file.<br> 
Specify to listen on only one internet protocol. The SSH daemon listens for incoming connections over both IPv4 and IPv6 by default. Unless you need to SSH into your Linode using both protocols, disable whichever you do not need. This does not disable the protocol system-wide, it is only for the SSH daemon.<br><br>

Add this option as `AddressFamily` is usually  not in the `/etc/ssh/sshd_config` file by default. AddressFamily `inet` to listen only on IPv4 or `AddressFamily inet6` to listen only on IPv6.<br>
Change accordingly to yours needs:
```bash
echo 'AddressFamily inet' | sudo tee -a /etc/ssh/sshd_config
```
or
```bash
echo 'AddressFamily inet6' | sudo tee -a /etc/ssh/sshd_config
```
<!-- 
   ADD something more about ssh ports!
-->
Finally change ssh port from default 22 to something different:<br>
```bash
# What ports, IPs and protocols we listen for
Port 3000
```
In this example I'm going to use port 3000 for all ssh connections.
<br><br>
After that restart the SSH service to load the new configuration.

```bash
sudo service ssh restart
```
<br>
<!-- Fail2ban is to heavy - it needs to be replaced (I've removed it from the server)
<p align="center">***
<p align="center">**Fail2ban for SSH**<br>

Fail2ban is a log-parsing application that monitors system logs for symptoms of an automated attack on your server. When an attempted attack is discovered, Fail2ban will add a new rule to iptables, thus blocking the IP address of the attacker (for a set amount of time or permanently - up to you really).<br>

Install Fail2ban and iptables:
```bash
emerge -av net-analyzer/fail2ban net-firewall/iptables net-firewall/ipset
```

Iptables should be already installed - it is only to make sure that we have it in the system. We'll configure it a bit later.<br><br>
For now go to `/etc/fail2ban`. Within this directory are all Fail2ban configuration files:<br>

```bash
cd /etc/fail2ban
```
and check configuration files there:
```bash
ls -l 
```
The default fail2ban configuration file is location at /etc/fail2ban/jail.conf. The configuration work should not be done in that file, however, and we will instead make a local copy of it.<br>

```bash
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

After the file is copied, you can make all of your changes within the new `jail.local file`. Many of possible services that may need protection are in the file already. Each is located in its own section, configured and turned off by default.<br>

Edit `jail.local` file:<br>

```bash
nano /etc/fail2ban/jail.local
```
The first section of defaults covers the basic rules that fail2ban will follow. If you want to set up more nuanced protection for your virtual private server, you can customize the details in each section.<br>
Write your personal IP address into the `ignoreip` line. You can separate each address with a space. IgnoreIP allows you white list certain IP addresses and make sure that they are not locked out from your server. Including your address will guarantee that you do not accidentally ban yourself.<br> 

```bash
[DEFAULT]

# "ignoreip" can be an IP address, a CIDR mask or a DNS host. Fail2ban will not
# ban a host which matches an address in this list. Several addresses can be
# defined using space separator.
ignoreip = 127.0.0.1

# "bantime" is the number of seconds that a host is banned.
bantime  = 3600

# A host is banned if it has generated "maxretry" during the last "findtime"
# seconds.
findtime  = 600

# "maxretry" is the number of failures before a host get banned.
maxretry = 3
```
`Bantime` parameter is the number of seconds that a host would be blocked from the server if they are found to be in violation of any of the rules. This is especially useful in the case of various bots, that once banned, will simply move on to the next target. The default is set for 10 minutes—you may raise this to an hour (3600sec or even higher).<br>

`Maxretry` parameter is the amount of incorrect login attempts that a host may have before they get banned for the length of the ban time.<br>

`Findtime` parameter refers to the amount of time that a host has to log in. The default setting is 10 minutes; this means that if a host attempts, and fails, to log in more than the maxretry number of times in the designated 10 minutes, they will be banned. Sweet - isn't it? :) <br>
<br>
Now lets check and configure the ssh-iptables section in the `/etc/fail2ban/jail.local` file.<br>
The SSH details section is by default already set up and turned on. Although you should not be required to make any changes within this section, you can find the details:<br>

```bash
[ssh-iptables]

enabled  = true
filter   = sshd
action   = iptables[name=SSH, port=3000, protocol=tcp]
logpath  = /var/log/secure
maxretry = 3
```

enabled - means that SSH protection is on. To turn it off use word: "false".<br>
filter - is set by default to sshd, and it refers to the config file containing the rules that fail2banuses to find matches.<br>
action - describes the steps that fail2ban will take to ban a matching IP address.<br>

In the "iptables" parameter's details, you can customize fail2ban a bit further. As we are going to use a non-standard port for ssh connection, we need to set the port number within the brackets to match.<br>

You can change the protocol from TCP to UDP in this line as well, depending on which one you want fail2ban to monitor.<br>

log path - this parameter refers to the log location that fail2ban will use.<br>

max retry - I'm sure I don't need to explain that one- right?<br>

When all changes in the fail2ban configurations are set and saved, there is only one thing left to do. Restart the fail2ban service to catch up with all changes:<br>

```bash
sudo service fail2ban restart
```
<br><br>
-->

<p align="center"><b>***</b></p>
<p align="center"><b>IPTABLES:</b></p>
<br>

It is time for the iptables configuration.<br>
There are few things to do, before we install it. First go and edit `/etc/portage/package.use`:
```bash
www-servers/nginx aio http http2 http-cache ipv6 nginx_modules_http_autoindex nginx_modules_http_browser nginx_modules_http_empty_gif nginx_modules_http_fancyindex nginx_modules_http_gzip ssl http_v2_module ngx_http_v2_module nginx_modules_image_filter ngx_http_empty_gif nginx_modules_http_referer nginx_modules_http_geo nginx_modules_http_geoip nginx_modules_http_gunzip nginx_modules_http2

# required by media-gfx/graphviz-2.26.3-r4::gentoo
# required by @preserved-rebuild (argument)
>=media-libs/gd-2.0.35-r4 truetype fontconfig

# If it doesn't work change to temporary fix (old version doesn't have hpn support): -hpn
# But current openssh works just fine.
net-misc/openssh hpn

# iptables geoip:
net-firewall/iptables extensions

# ipset for non modular kernel:
net-firewall/ipset -modules
# required by www-servers/nginx-1.10.1::gentoo[nginx_modules_http_image_filter]
# required by @selected
# required by @world (argument)
>=media-libs/gd-2.2.3 png jpeg
```
Lets install it:
<!--```bash
emerge -av net-analyzer/fail2ban net-firewall/iptables net-firewall/ipset
```
-->
```bash
emerge -av net-firewall/iptables net-firewall/ipset
```
To check the rules that iptables puts in effect religiously memorise the following command:<br>

```bash
iptables -L
```

Iptables is the controller for netfilter, the Linux kernel’s packet filtering framework. Iptables is included in most Linux distributions by default but is considered an advanced method of firewall control.<br>

Appropriate firewall rules depend on the services being run. Below are iptables rulesets to secure your web server.<br>

Setup firewall rules in a new file:<br>
```bash
sudo nano /etc/iptables.firewall.rules
```

The following firewall rules will allow `HTTP (80)`, `HTTPS (443)`, `SSH (3000)`, `ping`, and some other ports for testing. All other ports will be blocked.<br>

Paste the following into `/etc/iptables.firewall.rules`:

```bash
*filter

#  Allow all loopback (lo0) traffic and drop all traffic to 127/8 that doesn't use lo0
-A INPUT -i lo -j ACCEPT
-A INPUT ! -i lo -d 127.0.0.0/8 -j REJECT

#  Accept all established inbound connections
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Block anything from China
# These rules are pulled from ipset's china list
# The source file is at /etc/cn.zone (which in turn is generated by a shell script at /etc/block-china.sh )
-A INPUT -p tcp -m set --match-set china src -j DROP

#  Allow all outbound traffic - you can modify this to only allow certain traffic
-A OUTPUT -j ACCEPT

#  Allow HTTP and HTTPS connections from anywhere (the normal ports for websites and SSL).
-A INPUT -p tcp --dport 80 -j ACCEPT
-A INPUT -p tcp --dport 443 -j ACCEPT

#  Allow SSH connections
#  The -dport number should be the same port number you set in sshd_config
#  And the ssh port used in this example is: 3000.
#
-A INPUT -p tcp -m state --state NEW --dport 3000 -j ACCEPT

#  Allow ping
-A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT

# Fancy to install IRC maybe? (if not hash it):
-A INPUT -i eth0 -p tcp -m tcp --sport 6667 -j ACCEPT
-A OUTPUT -o eth0 -p tcp -m tcp --dport 6667 -j ACCEPT

#  Log iptables denied calls
-A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7

#  Drop all other inbound - default deny unless explicitly allowed policy
-A INPUT -j DROP
-A FORWARD -j DROP

# If you want to show to attackers/sniffers/bots that you are tired of their drama you can then reject instead of droping all other 
# inbounds. 
# Default deny unless explicitly allowed policy:
# -A INPUT -j REJECT
# -A FORWARD -j REJECT

COMMIT
```
Yep. Rule `-A INPUT -p tcp -m set --match-set china src -j DROP` is a brilliant thing. Nothing personal guys.<br>
I was tired of all awkward traffic from china. Actually not even a single website I'm hosting is directed for that region. I'm even thinking to extend it a bit for Russia and Ukraine  as well. Websites I'm hosting usually if not always are for local makrket or a single country only. Nothing 'THAT BIG' for a global reach or something like that (lets be serious though - this is one small VPS). The amount of sniffers, bots and bad IPs from China, Russia, Ukraine and Romania is teryfying. Hundreds to thousands daily.. :/ For a small servers bots trying to get in through the ssh port:22 every 3 seconds or sniffers looking for security hole in particular software (mysql and php related usually) it is nuisance and waste of computing resources.<br>

Lets build those rules now.<br>

Start with creating a following file:

```bash
nano /etc/block-china.sh
```

And inside of that file write the following:

```bash
# Create the ipset list
ipset -N china hash:net
# remove any old list that might exist from previous runs of this script
rm cn.zone
# Pull the latest IP set for China
wget -P . http://www.ipdeny.com/ipblocks/data/countries/cn.zone
# Add each IP address from the downloaded list into the ipset 'china'
for i in $(cat cn.zone ); do ipset -A china $i; done
# Restore iptables
/sbin/iptables-restore < /etc/iptables.firewall.rules
```

Make sure the rules are in place after any reboot:

```bash
nano /etc/network-iptables-rules
```

Inside of that file put:

```bash
#!/bin/sh
/sbin/iptables-restore < /etc/iptables.firewall.rules
```

<!-- TODO: That is not working yet!
     RULES won't restore back on gentoo - this method require manual restore somehow - to review! :/ 
-->

Set the script permissions:

```bash
sudo chmod +x /etc/network-iptables-rules
```

Before we go any further there is a need to install one more thing:

```bash
emerge -av net-firewall/ipset
```
Coutry based rules in IPTABLES won't work without IPSET. So we need IPSET to smartly block some countries. :/ <br>

Restore IPTABLES rules now to make them filter all network traffic:

```bash
iptables-restore < /etc/iptables.firewall.rules
```

Verify that the rules were installed correctly:

```bash
sudo iptables -vL
```

If you are using IPv6 protocol just repeat process for ip6tables same as shown above. The only difference will be usage of `ip6tables` instead of `iptables`.

Save all iptables rules for now:

```bash
service iptables save
```

I know that we have barely scratched the surface with securing this tiny fluffy server. But it is good enough for now (cough! cough!..). I will cover security topic at the end of this documentation (when all services are set, tested and running). <br>

Lets install some basic services for hosting websites, irc and git. As we are focusing on keeping it speed and secure there is no place for databases or things like php nasty bloat.
<br><br>
<p align="center"><b>***</b></p>
<p align="center"><b>Web Server:</b></p>
<br>

<p align="left">While I am not aware of any Gentoo specific benchmarks that have been run on the various flavors of web servers, it is typically logical to assume that the more feature-enabled by default web server you install, the more resources it would use.<br>
I spent a lot of time reading various documentations and comparing all informations carefully. I also had in the past my ups and downs with Apache, I did some trials with Lighttpd, etc.. Long story short NGINX web server is better in the 'raw number of requests per second it can serve' than Apache or Lighttpd servers no matter how well configured they are. At higher levels of concurrency, NGINX can handle fewer requests per second, but still can serve double what Lighttpd does (which is already doing nearly 4x what Apache was ever able to do).</p>
<br>
<p align="center">
  <img src="https://github.com/rkruk/gentoo-server-setup/blob/master/Webserver_requests_graph.jpg">
</p>
<br>
<p align="left">Though Apache supports a larger toolbox of things it can do immediately after install, yet it can be something of a memory hog with all modules enabled by default. Contrary freshly installed NGINX does not eat as much memory compared to Apache (even with enabled additional modules is still faster and scale way better).<br><br>
I should also mention that nginx is a also good as a reverse proxy server!<br><br>

Before we go with installation of nginx package, first we should do some magic with the USE flags for Nginx.<br>
There are two ways of handling USE flags:<br><br>
-Global (via /etc/portage/make.conf file - more information <a href="https://wiki.gentoo.org/wiki//etc/portage/make.conf">here</a>)<br>
-Local (via /etc/portage/package.use - more <a target="_blank" href="https://wiki.gentoo.org/wiki//etc/portage/package.use">here</a>)
<br><br>
Nginx uses modules to enhance its features. It uses expanded USE (USE_EXPAND) flags to denote which modules should be installed:<br>
-HTTP related modules can be enabled through the NGINX_MODULES_HTTP variable<br>
-Mail related modules can be enabled through the NGINX_MODULES_MAIL variable<br>
-Third party modules can be enabled through the NGINX_ADD_MODULES variable<br>
<br>
These variables need to be set in /etc/portage/make.conf. Modules detailed descriptions can be found in these directories:<br> `/usr/portage/profiles/desc/nginx_modules_http.desc`<br> 
and<br> 
`/usr/portage/profiles/desc/nginx_modules_mail.desc.`
<br><br>
For example, to enable the spdy module:<br>
Edit `/etc/portage/make.conf` and add:
```bash
NGINX_MODULES_HTTP="spdy"
```
That will overwrite the default value of NGINX_MODULES_HTTP and set it to spdy. To enable the spdy without overwriting the default NGINX_MODULES_HTTP, use local USE flag which can be specified in `/etc/portage/package.use`:<br>
```bash
www-servers/nginx NGINX_MODULES_HTTP: spdy
```
You definitely should check the complete list of USE flags specified for <a target="_blank" href="https://packages.gentoo.org/packages/www-servers/nginx">www-servers/nginx</a> package.<br>
Ask what flags nginx have enabled by default:
```bash
equery uses nginx
```
And change flags accordingly to your needs in the `/etc/portage/make.conf`. For starters it should look like:<br>
```bash
USE="aio bindist http http-cache http2 ipv6 mmx pcre poll select sse sse2 ssl threads -cpp_test -debug -google_perftools"

NGINX_MODULES_HTTP="autoindex browser charset empty_gif geo gzip limit_conn limit_req map proxy referer rewrite scgi split_clients ssi upstream_hash upstream_ip_hash upstream_keepalive upstream_least_conn upstream_zone userid uwsgi geoip gunzip gzip_static image_filter realip"
```
I'm sure that portage with throw some errors here and there.

```bash

```
<br>
With all USE flags set, lets install www-servers/nginx:
```bash
emerge --ask www-servers/nginx
```
The nginx package will install an init service script allowing you to stop, start, or restart the service.<br> 
For example to start the nginx service do:
```bash
service nginx start
```
To restart do:
```bash
service nginx restart
```
You get that? ;) 

<b>BUT DON'T START NGINX YET!!</b>
<br>
There is a lot to do with nginx configuration before firing it up. Let's start with changing a few files in the `/etc/nginx/` directory. 
Our points of interest for now are `mime.types`  `nginx.conf` files. There is one more configuration inside `/etc/nginx/sites-available' - lets call it `example.com` file. As you can guess it is responsible of handling `example.com` domain.<br>

The main nginx configuration is handled through the `/etc/nginx/nginx.conf` file. I'm using Cloudflare DNS, GeoIP exlusions, Gzip, and I'm focusing here on delivering websites written in pure html, css, js (no php, ruby, rust, js frameworks or other weird things cool kids use nowadys). Remember it is all about performance and simplicity here. No over complication with containers, markup conversions - just html websites. Simple as that. Lets tune main nginx configuration to squize a bit more from it:<br>

```bash
nano /etc/nginx/nginx.conf
```
And do some changes here:<br>

```bash
#
user nginx nginx;
# It is common practice to run 1 worker process per core. Anything above this won't hurt your system, 
# but it will leave idle processes usually just lying about.
# To figure out what number you'll need to set worker_processes to, simply take a look at the amount 
# of cores you have on your setup: grep processor /proc/cpuinfo | wc -l
# In this example I'm using small one core CPU VPS:
worker_processes 1;

error_log /var/log/nginx/error_log info;

events {
        # The worker_connections command tells our worker processes how many people can simultaneously 
        # be served by Nginx. The default value is 768; however, considering that every browser usually 
        # opens up at least 2 connections/server, this number can half. To check our core's limitations
        # issue: ulimit -n and use the result of this command as a good starting point.
        worker_connections 1024;
        use epoll;
        # for a worker to accept all new connections at one time:
        multi_accept on;
}

http {
        ### Cloudflare Real IP:
        # I know what people talking about some particular companies ;)
        # I also know that Cloudflare is blocking TOR nodes - which is a terrible idea. 
        # People also are a bit worried about their FlexibleSSL which I'm not recommending (use fullSSL or Let’s Encrypt SSL). 
        # To be honest - their DNS package is quite nice for me. They even offer some candy there: 
        # CDN, statistics (not great but always something), free SSL for my small websites, and some
        # sort protection agains DDOS. And it is dead easy to set up quickly. Not a Cloudflare user? You can skip this part:
        set_real_ip_from   204.93.240.0/24;
        set_real_ip_from   204.93.177.0/24;
        set_real_ip_from   199.27.128.0/21;
        set_real_ip_from   173.245.48.0/20;
        set_real_ip_from   103.22.200.0/22;
        set_real_ip_from   141.101.64.0/18;
        set_real_ip_from   108.162.192.0/18;
        set_real_ip_from   103.21.244.0/22;
        set_real_ip_from   103.31.4.0/22;
        set_real_ip_from   104.16.0.0/12;
        set_real_ip_from   131.0.72.0/22;
        set_real_ip_from   162.158.0.0/15;
        set_real_ip_from   172.64.0.0/13;
        set_real_ip_from   188.114.96.0/20;
        set_real_ip_from   190.93.240.0/20;
        set_real_ip_from   197.234.240.0/22;
        set_real_ip_from   198.41.128.0/17;
        # IPV6 Cloudflare IP:
        set_real_ip_from   2400:cb00::/32;
        set_real_ip_from   2405:8100::/32;
        set_real_ip_from   2405:b500::/32;
        set_real_ip_from   2606:4700::/32;
        set_real_ip_from   2803:f800::/32;
        real_ip_header     CF-Connecting-IP;
        ### End of Cloudflare Real IP list ###

        ### My little experiment with GeoIP.
        # The list is long as I don't believe in global reach for my small websites I'm hosting on this particular VPS. 
        # Viewer's audience is targeted here for a few countries only. I'm hoping also to cut the drama with spammers, 
        # and others not so happy folks on the internet.
        #
        geoip_city    /etc/nginx/geoip/GeoLiteCity.dat;
        geoip_country /usr/share/GeoIP/GeoIP.dat;
        map $geoip_country_code $allowed_country {
                default yes;
                # Block those countries:
                AD no;
                AE no;
                AF no;
                AG no;
                AI no;
                AL no;
                AM no;
                AO no;
                AP no;
                AR no;
                AS no;
                AW no;
                AX no;
                AZ no;
                BA no;
                BB no;
                BD no;
                BF no;
                BG no;
                BH no;
                BI no;
                BJ no;
                BL no;
                BM no;
                BN no;
                BO no;
                BQ no;
                BR no;
                BS no;
                BT no;
                BV no;
                BW no;
                BY no;
                BZ no;
                CC no;
                CD no;
                CF no;
                CG no;
                CI no;
                CK no;
                CL no;
                CM no;
                CN no;
                CO no;
                CR no;
                CU no;
                CV no;
                CW no;
                CX no;
                CY no;
                DJ no;
                DK no;
                DM no;
                DO no;
                DZ no;
                EC no;
                EE no;
                EG no;
                EH no;
                ER no;
                ET no;
                FI no;
                FJ no;
                FK no;
                FM no;
                GA no;
                GD no;
                GE no;
                GF no;
                GG no;
                GH no;
                GI no;
                GM no;
                GN no;
                GP no;
                GQ no;
                GR no;
                GS no;
                GT no;
                GU no;
                GW no;
                GY no;
                HK no;
                HM no;
                HN no;
                HR no;
                HT no;
                HU no;
                ID no;
                IN no;
                IO no;
                IQ no;
                IR no;
                JM no;
                JO no;
                JP no;
                KE no;
                KG no;
                KH no;
                KI no;
                KM no;
                KN no;
                KP no;
                KR no;
                KW no;
                KZ no;
                LA no;
                LB no;
                LC no;
                LK no;
                LR no;
                LS no;
                LT no;
                LV no;
                LY no;
                MA no;
                MD no;
                MG no;
                MK no;
                ML no;
                MM no;
                MN no;
                MQ no;
                MR no;
                MT no;
                MU no;
                MV no;
                MW no;
                MX no;
                MY no;
                MZ no;
                NA no;
                NC no;
                NE no;
                NG no;
                NI no;
                NP no;
                NR no;
                NU no;
                NZ no;
                OM no;
                PA no;
                PE no;
                PF no;
                PG no;
                PH no;
                PK no;
                PM no;
                PN no;
                PR no;
                PS no;
                PT no;
                PW no;
                PY no;
                QA no;
                RE no;
                RO no;
                RS no;
                RU no;
                RW no;
                SA no;
                SB no;
                SC no;
                SD no;
                SG no;
                SI no;
                SJ no;
                SL no;
                SN no;
                SO no;
                SR no;
                SS no;
                ST no;
                SV no;
                SY no;
                SZ no;
                TC no;
                TD no;
                TG no;
                TH no;
                TJ no;
                TK no;
                TL no;
                TM no;
                TN no;
                TO no;
                TR no;
                TT no;
                TV no;
                TW no;
                TZ no;
                UA no;
                UG no;
                UY no;
                UZ no;
                VC no;
                VE no;
                VG no;
                VI no;
                VN no;
                VU no;
                WF no;
                WS no;
                YE no;
                YT no;
                ZA no;
                ZM no;
                ZW no;
        }
        # geo $exclusions {
        # default 0;
        # 10.8.0.0/24 1;
        # }
        #
        # About those Geoip settings - I'm not sure if that thing is working at all :/
        # I'll definitely look closer at this when I have a time for that.

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        log_format main
                '$remote_addr - $remote_user [$time_local] '
                '"$request" $status $bytes_sent '
                '"$http_referer" "$http_user_agent" '
                '"$gzip_ratio"';

        # client_body_buffer_size handles the client buffer size.
        # Most client buffers are coming from POST method form submissions.
        # 128k is normally a good choice for this setting.
        # A bit overkill I know..
        # If you are preparing yourself for buffer overflows scenario set it up to: 1k.
        # As I'm serving it through the Cloudflare networked proxy:
        client_body_buffer_size      128k;

        # client_max_body_size sets the max body buffer size.
        # If the size in a request exceeds the configured value,
        # the 413 (Request Entity Too Large) error is returned to the client.
        # For reference, browsers cannot correctly display 413 errors.
        # Setting size to 0 disables checking of client request body size.
        # I could go for 1k here but lets try for a bit 10m:
        client_max_body_size         10m;

        # client_header_buffer_size handles the client header size.
        # 1k is usually a sane choice for this by default.
        client_header_buffer_size    1k;

        # large_client_header_buffers shows the maximum number and size of
        # buffers for large client headers. 4 headers with 4k buffers should be sufficient here.
        large_client_header_buffers  4 4k;

        # output_buffers sets the number and size of the buffers used for reading a response from a disk.
        # If possible, the transmission of client data will be postponed until Nginx has at least the
        # set size of bytes of data to send. The zero value disables postponing data transmission.
        output_buffers               1 32k;
        postpone_output              1460;

        # client_header_timeout sends directives for the time a server will wait for a header body to be sent.
        client_header_timeout 3m;

        # client_body_timeout sends directives for the time a server will wait for a body to be sent.
        client_body_timeout 3m;

        # sent_timeout specifies the response timeout to the client.
        # This timeout does not apply to the entire transfer but, rather,
        # only between two subsequent client-read operations. Thus, if
        # the client has not read any data for this amount of time, then Nginx shuts down the connection.
        send_timeout 3m;

        # configuration block tells Nginx to cache 1000 files for 30 seconds,
        # excluding any files that haven't been accessed in 20 seconds, and only files that have 5 times or more.
        open_file_cache max=1000 inactive=20s;
        open_file_cache_valid 30s;
        open_file_cache_min_uses 5;
        open_file_cache_errors off;

        # cache via a particular location <-- pushed to: /sites-available/*conf:
        #location ~* .(woff|eot|ttf|svg|mp4|webm|jpg|jpeg|png|gif|ico|css|js)$ {
        #    expires 365d;
        #    }

        connection_pool_size 256;
        request_pool_size 4k;

        # disable sending the nginx version number in error pages and Server header:
        server_tokens off;

        # Gzip settings:
        gzip on;
        gzip_disable "MSIE [1-6]\.";
        gzip_vary on;
        gzip_proxied any;
        gzip_comp_level 6;
        gzip_buffers 16 8k;
        gzip_http_version 1.1;
        gzip_min_length 256;
        gzip_types application/x-javascript text/css application/javascript text/javascript text/plain text/xml application/json application/vnd.ms-fontobject application/x-font-opentype application/x-font-truetype application/x-font-ttf application/xml font/eot font/opentype font/otf image/svg+xml image/vnd.microsoft.icon image/x-icon;

        gzip_proxied  expired no-cache no-store private auth;
        # End of gzip settings

        # sendfile optimizes serving static files from the file system, like logos:
        sendfile on;

        # tcp_nopush optimizes the amount of data sent down the wire at once by activating
        # the TCP_CORK option within the TCP stack. TCP_CORK blocks the data until the packet
        # reaches the MSS, which is equal to the MTU minus the 40 or 60 bytes of the IP header.
        tcp_nopush on;

        # tcp_nodelay allows Nginx to make TCP send multiple buffers as individual packets:
        tcp_nodelay on;

        # Keep alive allows for fewer reconnections from the browser:
        keepalive_timeout 75 20;
        keepalive_requests 100000;

        ignore_invalid_headers on;

        index index.htmL;

        include /etc/nginx/sites-enabled/*.conf;
}
```
<br>
Inside of `nginx.conf` there are a few things we need to cover now before anything else. There is a GeoIP module to install and set up. The <a target="_blank" href="http://dev.maxmind.com/geoip/geoip2/geolite2/">GeoLite</a> databases are distributed under the Creative Commons Attribution-ShareAlike 4.0 International License by <a target="_blank" href="http://www.maxmind.com">http://www.maxmind.com</a>.

```bash 
emerge -Ss sys-process/dcron dev-libs/geoip net-misc/geoipupdate
```
This will install "geoiplookup" and "geoipupdate" to update the database.<br>
As the geoiplookup database will be pretty outdated you might want to update it regularly as IP assignment changes. One way of doing that is to use a crontab. Get that database installed. First country rules:
```bash
wget -q https://geolite.maxmind.com/download/geoip/database/GeoLite2-Country.tar.gz
-O - | gunzip > /etc/nginx/geoip/GeoIP.dat.new && mv /etc/nginx/geoip/GeoIP.dat.new /etc/nginx/geoip/GeoIP.dat 
```
<br>
Then city rules (because why not?):
```bash
mkdir /etc/nginx/geoip && cd /etc/nginx/geoip && wget -N https://geolite.maxmind.com/download/geoip/database/GeoLite2-City.tar.gz && gunzip GeoLiteCity.dat.gz

```
To update it regularly create a new file in the cron.monthly:
```bash
nano /etc/cron.monthly/GeoIP
```
Inside of that file put that script:
```bash
#!/bin/bash
wget -q https://geolite.maxmind.com/download/geoip/database/GeoLite2-City.tar.gz -O - | gunzip > /etc/nginx/geoip/GeoCity.dat.new && mv /usr/share/geoip/GeoCity.dat.new /usr/share/geoip/GeoCity.dat
wget -q https://geolite.maxmind.com/download/geoip/database/GeoLite2-Country.tar.gz -O - | gunzip > /etc/nginx/geoip/GeoIP.dat.new && mv /etc/nginx/geoip/GeoIP.dat.new /etc/nginx/geoip/GeoIP.dat
```
Now we can start our freshly installed `dcron`:
```bash
/etc/init.d/dcron start
rc-update add dcron default
```
GeoIP is set and running.<br>
More informations about GeoIP datasets are <a target="_blank" href="https://dev.maxmind.com/geoip/geoip2/geolite2/">here</a>.
<br>
Lets install something lightweight to rotate system logs:
```bash 
emerge -av app-admin/metalog
rc-update add metalog default
```
It looks a bit chaotic.. - but don't worry :). A few final touches with nginx and we are done here. 
In the `/etc/nginx/nginx.conf` file - there is `mime.types` line - right? Lets check if we have it as it should be:

```bash
nano /etc/nginx/mime.types
```
Inside of `mime.types` we can specify exactly what types of file nginx can serve. Here is a comprehensive list of all files you can include inside `mime.types`:
```bash
types {
    text/html                             html htm shtml;
    text/css                              css;
    text/xml                              xml;
    image/gif                             gif;
    image/jpeg                            jpeg jpg;
    application/javascript                js;
    application/atom+xml                  atom;
    application/rss+xml                   rss;

    text/mathml                           mml;
    text/plain                            txt;
    text/vnd.sun.j2me.app-descriptor      jad;
    text/vnd.wap.wml                      wml;
    text/x-component                      htc;

    image/png                             png;
    image/tiff                            tif tiff;
    image/vnd.wap.wbmp                    wbmp;
    image/x-icon                          ico;
    image/x-jng                           jng;
    image/x-ms-bmp                        bmp;
    image/svg+xml                         svg svgz;
    image/webp                            webp;

    application/font-woff                 woff;
    application/java-archive              jar war ear;
    application/json                      json;
    application/mac-binhex40              hqx;
    application/msword                    doc;
    application/pdf                       pdf;
    application/postscript                ps eps ai;
    application/rtf                       rtf;
    application/vnd.apple.mpegurl         m3u8;
    application/vnd.ms-excel              xls;
    application/vnd.ms-fontobject         eot;
    application/vnd.ms-powerpoint         ppt;
    application/vnd.wap.wmlc              wmlc;
    application/vnd.google-earth.kml+xml  kml;
    application/vnd.google-earth.kmz      kmz;
    application/x-7z-compressed           7z;
    application/x-cocoa                   cco;
    application/x-java-archive-diff       jardiff;
    application/x-java-jnlp-file          jnlp;
    application/x-makeself                run;
    application/x-perl                    pl pm;
    application/x-pilot                   prc pdb;
    application/x-rar-compressed          rar;
    application/x-redhat-package-manager  rpm;
    application/x-sea                     sea;
    application/x-shockwave-flash         swf;
    application/x-stuffit                 sit;
    application/x-tcl                     tcl tk;
    application/x-x509-ca-cert            der pem crt;
    application/x-xpinstall               xpi;
    application/xhtml+xml                 xhtml;
    application/xspf+xml                  xspf;
    application/zip                       zip;

    application/octet-stream              bin exe dll;
    application/octet-stream              deb;
    application/octet-stream              dmg;
    application/octet-stream              iso img;
    application/octet-stream              msi msp msm;

    application/vnd.openxmlformats-officedocument.wordprocessingml.document    docx;
    application/vnd.openxmlformats-officedocument.spreadsheetml.sheet          xlsx;
    application/vnd.openxmlformats-officedocument.presentationml.presentation  pptx;

    audio/midi                            mid midi kar;
    audio/mpeg                            mp3;
    audio/ogg                             ogg;
    audio/x-m4a                           m4a;
    audio/x-realaudio                     ra;

    video/3gpp                            3gpp 3gp;
    video/mp2t                            ts;
    video/mp4                             mp4;
    video/mpeg                            mpeg mpg;
    video/quicktime                       mov;
    video/webm                            webm;
    video/x-flv                           flv;
    video/x-m4v                           m4v;
    video/x-mng                           mng;
    video/x-ms-asf                        asx asf;
    video/x-ms-wmv                        wmv;
    video/x-msvideo                       avi;
}
```

<br>
And that is all. The neccessary basics to do with the nginx core settings are mostly done. Now it is time to set up configuration for some websites. :D<br>
It is recommended to configure each website separately in the `/etc/nginx/sites-available/` and link it to the `/etc/nginx/sites-enabled/` directory.<br> 
Now lets edit `/etc/nginx/sites-available/example.conf` file (change 'example' and use your own domain name here to know what is where later).<br>

```bash
nano /etc/nginx/sites-available/example.conf
```
Inside of that file we need to put a lot of informations. Remember that we are going for a performance savy configuration here. In examples below I'll show you my two 'optimal' configurations I use for various websites.
<br>
<br>
Let's try something smaller at the beginning (you can extend it later):<br><br>
<p align="center"><b>VERSION 1</b>(recommended):<br>
```bash
server {
    listen 80;
    server_name www.example.com;
    return 301 $scheme://example.com$request_uri;
    }

server {
    listen 80;
    server_name example.com;

    root /home/user/website/example;

    index index.html index.htm;

    charset utf-8;

    location / {
      index index.html index.htm;
      autoindex on;
    }

    gzip on;
    gzip_min_length 1000;
    gzip_proxied  expired no-cache no-store private auth;
    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
    
    access_log /var/log/nginx/localhost.access_log main;
    error_log /var/log/nginx/error_log info;
    
    location ~* \.html$ {
      expires -1;
    }

    location ~*  \.(jpg|jpeg|png|gif|ico|css|js|xml)$ {
      access_log off;
      log_not_found off;
      expires 30d;
      add_header Pragma public;
      add_header Cache-Control "public, must-revalidate, proxy-revalidate";
    }
}
```
<br>
<br>
<p align="center"><b>VERSION 2</b> (extended):<br> 
```bash
# www to no www:
server {
       listen 80;
       server_name www.example.com;
       return 301 $scheme://example.com$request_uri;
}

# no www
server {
       # Listen on both IPv4 and IPv6
       listen 443 ssl; # default deferred;
       listen [::]:443 ipv6only=on ssl; # default deferred;

       server_name example.com;

       # Hide nginx version:
       server_tokens off;

       # SSL Cert location:
       ssl_certificate /etc/nginx/ssl/example.com.pem;
       # example.com.pem - KEY -important as hell!
       ssl_certificate_key /etc/nginx/ssl/example.com.key;

       # enable session resumption to improve https performance
       # https://vincent.bernat.im/en/blog/2011-ssl-session-reuse-rfc5077.html
       ssl_session_cache shared:SSL:50m;
       ssl_session_timeout 5m;

       # Diffie-Hellman parameter for DHE ciphersuites, recommended 2048 bits.
       ssl_dhparam /etc/nginx/ssl/dhparam.pem;

       # enables server-side protection from BEAST attacks
       # https://blog.ivanristic.com/2013/09/is-beast-still-a-threat.html
       ssl_prefer_server_ciphers on;

       # disable SSLv3(enabled by default since nginx 0.8.19) since it's less secure then TLS
       # https://en.wikipedia.org/wiki/Secure_Sockets_Layer#SSL_3.0
       ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

       # ciphers chosen for forward secrecy and compatibility
       # https://blog.ivanristic.com/2013/08/configuring-apache-nginx-and-openssl-for-forward-secrecy.html
       ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:ECDHE-RSA-RC4-SHA:ECDHE-ECDSA-RC4-SHA:RC4-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!3DES:!MD5:!PSK';
       # enable ocsp stapling (mechanism by which a site can convey certificate revocation information to visitors in a 
       # privacy-preserving, scalable manner)
       # --> https://blog.mozilla.org/security/2013/07/29/ocsp-stapling-in-firefox/

       # SSL Stapling(Doesn't work on cloudflare SSl Cert (no CA Cert Available)):
       # ssl_trusted_certificate /etc/nginx/ssl/..??;
       # resolver 8.8.8.8;
       # ssl_stapling on;
       # ssl_stapling_verify on;

       # config to enable HSTS(HTTP Strict Transport Security) 
       # --> https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security
       # to avoid ssl stripping https://en.wikipedia.org/wiki/Moxie_Marlinspike#SSL_stripping
       add_header Strict-Transport-Security "max-age=31536000; includeSubdomains;";

       # And here is the rest of the configuration.
       # Change this location accordingly to the location where you keep your website:
       root /home/user/website/example;

       index index.html index.htm;

       # Disable unwanted HTTP Methods:
       if ($request_method !~ ^(GET|HEAD|POST)$) {
          return 444;
          }

       # GeoIP
       if ($allowed_country = no) {
            return 444;
        }
       # End of GeoIP

       charset utf-8;

       # Only Cloudflare traffic through nginx:
       location / {
          index index.html index.htm;
          try_files $uri $uri/ /index.html;
          autoindex on;
          }

       gzip on;
       gzip_min_length 1000;
       gzip_proxied  expired no-cache no-store private auth;
       gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

       error_page 404 /404.html;

       access_log /var/log/nginx/localhost.access_log main;
       error_log /var/log/nginx/error_log info;

       location /404.html {
          # Change it accordingly to the location of website files:
          root /home/user/website/example;
          }

       location ~* \.html$ {
          expires -1;
          }

       location ~*  \.(jpg|jpeg|png|gif|ico|css|js|xml)$ {
          access_log off;
          log_not_found off;
          expires 30d;
          add_header Pragma public;
          add_header Cache-Control "public, must-revalidate, proxy-revalidate";
        }

       # Image hotlinking protection:
       location ~ \.(jpg|jpeg|png|gif|ico|jpe?g)$ {
          valid_referers none blocked example.com *.example.com;
          if ($invalid_referer) {
             return   403;
          }
         } 

       location ~* \.(gif|jpg|jpeg|png|wmv|ico|avi|mpg|mpeg|mp4|htm|html|js|css)$ {
         valid_referers none blocked example.com www.example.com ~\.google\. ~\.yahoo\. ~\.bing\. ~\.facebook\. ~\.fbcdn\.;
           if ($invalid_referer) {
               return 403;
                 }
       }

       # Cache settings for static files directory:
       location ~* /images/.+\.[0-9a-f][0-9a-fA-F][0-9a-fA-F][0-9a-fA-F][0-9a-fA-F][0-9a-fA-F][0-9a-fA-F][0-9a-fA-F][0-9a-fA-F][0-9a-fA-F]\. {
       expires max;
       }
}

# no www to https
# redirect all http traffic to https
server {
       # Listen on both IPv4 and IPv6
       listen 80;
       listen [::]:80 ipv6only=on;
       server_name example.com;
       return 301 https://example.com$request_uri;
}
```
<br>
Start with the 'version 1' and configure it carefully. When you are happy with your results start adding changes from the 'version 2' 
<br><br>
When your `example.conf` is done and save - it is time to link it to the `/etc/nginx/sites-enabled` directory.<br>

```bash
ln -s /etc/nginx/sites-available/example.conf /etc/nginx/sites-enabled/aaaexample.conf
```
Above I've wrote `aaaexample.conf` to set which websites is set as the first and main for nginx server. There is no logic behind that you can name it as you want as long as you know what you are doing.<br><br>

All is set and in place (probably). Lets test it:<br>
```bash
nginx -t
```
Eliminate all errors from your configs. run `nginx -t` to the moment you have good working setup.<br> 
And push it online:<br>
```bash
service nginx start
```
<br>
Finally add nginx to the start:
```bash
rc-update add nginx default
```

<b>TO DO:</b>
<br>
- [ ] services,
- [ ] configuration of metalog, denyhosts, ntp daemon, etc..
- [ ] tests,
- [ ] apply tighter security.
