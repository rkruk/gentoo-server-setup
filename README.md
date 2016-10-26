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
<p align="center">****
<p align="center">**First Steps**

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
<p align="center">***
<p align="center">**Fail2ban for SSH**<br>

Fail2ban is a log-parsing application that monitors system logs for symptoms of an automated attack on your server. When an attempted attack is discovered, Fail2ban will add a new rule to iptables, thus blocking the IP address of the attacker (for a set amount of time or permanently - up to you really).<br>

Install Fail2ban and iptables:
```bash
emerge -av net-analyzer/fail2ban net-firewall/iptables
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
<p align="center">***
<p align="center">**IPTABLES:**
<br>

To check the rules that fail2ban puts in effect within the IP table religiously memorise and use following command:<br>

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
     RULES won't restore back - this method require manual restore somehow :/ 
-->

Set the script permissions:

```bash
sudo chmod +x /etc/network-iptables-rules
```

Restore those rules to make them filter all network traffic:

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
<p align="center">***
<p align="center">**Web Server:**
<br>

<p align="left">While I am not aware of any Gentoo specific benchmarks that have been run on the various flavors of web servers, it is typically logical to assume that the more feature-enabled version of web server you use, the more resources it would use.<br>
According to documentations I compared and readed carefully - NGINX web server is better in the 'raw number of requests per second it can serve' than Apache or Lighttpd servers. At higher levels of concurrency, NGINX can handle fewer requests per second, but still can serve double what Lighttpd does (which is already doing nearly 4x what Apache was ever able to do).<br>
<p align="center">![Gentoo](https://github.com/rkruk/gentoo-server-setup/blob/master/Webserver_requests_graph.jpg)<br>
<p align="left">Though Apache supports a larger toolbox of things it can do immediately after install, yet it can be something of a memory hog with all modules enabled by default. Contrary freshly installed NGINX does not eat as much memory compared to Apache (even with enabled additional modules is still faster and scale better).<br><br>
<b>TO DO:</b>
<br>
- [ ] services,
- [ ] configuration,
- [ ] tests,
- [ ] apply tighter security.
