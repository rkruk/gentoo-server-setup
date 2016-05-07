***
<p align="center">![Gentoo](https://github.com/rkruk/gentoo-server-setup/blob/master/larry.png)
<br>
#### <p align="center">**Gentoo Server Setup** 

<p align="center">Set Up your own Gentoo Web Server <br> or Cloud VPS 

------------------------------------------------------------------------

----------
<br>
<p align="center">**Introduction**

<p align="center">First of all let me state: <br>
**- I'm not insane**.
<br>
**- It is worth the effort**.
<br>
<br>
I have a few Gentoo based servers with a long uptime (not like it is a bad thing..) - and I bet there are working a way better than any Ubuntu, CentOS,etc.. servers I've used in the past.<br><br> *I know - this is a* **bold** *statement to say things like that. But let me explain*:
<p align="left">1. Continous uptime without the need of reboot to switch to the new version of the kernel, services,etc..
<p align="left">2. System is fully customisable, every single part can be adjusted as you like it, every daemon, service - you decide what you want - not the distro maintainers with tedious dependencies forcing you to use THAT version of software. Example: NTP daemon is quite resource heavy for a small daemon - I chenged it openntpd. Same thing with log damemons, web servers,.. 
<p align="left">3. Low resource footprint (really lower than anything else I have used in the past).<br><br>
While I am not aware of any Gentoo specific benchmarks that have been run on the various flavors of web servers, it is typically logical to assume that the more feature-enabled the version of web server you use, the more resources it would use.<br>
Though Nginx is better in the raw number of requests per second it can serve than Apache or Lighttpd. At higher levels of concurrency, it can handle fewer requests per second, but still can serve double what Lighttpd does (which is already doing nearly 4x what Apache was ever able to do).<br>
Though Apache supports a larger toolbox of things it can do immediately after install - it can be something of a memory whore with the more modules enabled. Nginx does not eat as much memory compared to Apache with enabled modules.<br>
