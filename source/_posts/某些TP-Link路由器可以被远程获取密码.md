---
title: 某些TP-Link路由器可以被远程获取密码
categories: 安全
tags:
  - 系统安全
date: 2014-01-19 17:59:38
---

# HOW I SAVED YOUR A** FROM THE ZYNOS (ROM-0) ATTACK !! ( FULL DISCLOSURE )

Hello everyone, I just wanted to discuss some vulnerability I found and exploited for GOODNESS .. just so that SCRIPT KIDIES won’t attack your home/business network .

Well, in Algeria the main ISP ( Algerie Telecom ) provide you with a router when you pay for an internet plan. So you can conclude that every subscriber is using that router . TD-W8951ND is one of them, I did some ip scanning and I found that every router is using ZYXEL embedded firmware.

# **Analysis :**

Let’s download an update and take a look at it and try to find some vulnerabilities. ( [http://www.tp-link.com/Resources/software/TD-W8951ND_V3.0_110729_FI.rar ](http://www.tp-link.com/Resources/software/TD-W8951ND_V3.0_110729_FI.rar))

[![lif](http://static.oschina.net/uploads/img/201401/19175934_qaNU.jpg "lif")](http://static.oschina.net/uploads/img/201401/19175934_8zu4.jpg)

The ras file is in LIF format !! …  
Hmmm let’s put that file for Binwalk test for God’s sake ! ( check :[http://code.google.com/p/binwalk/wiki/Installation](http://code.google.com/p/binwalk/wiki/Installation) for more informations on how to install it ).

This is what Binwalk told me about that file :  
[![zynos](http://static.oschina.net/uploads/img/201401/19175934_svCo.jpg "zynos")](http://static.oschina.net/uploads/img/201401/19175934_veKb.jpg)

You can clearly see and confirm that the router is using zynos firmware. We can also see that there is two blocks of LZMA compressed data … let’s extract them and have a look.  
[![extract1](http://static.oschina.net/uploads/img/201401/19175935_6pIO.jpg "extract1")](http://static.oschina.net/uploads/img/201401/19175935_0q2n.jpg)

The problem is that when I tried to decompress the two blocks I get an error : ” Compressed data is corrupt “

[![error](http://static.oschina.net/uploads/img/201401/19175935_U9nN.jpg "error")](http://static.oschina.net/uploads/img/201401/19175935_xf7b.jpg)

Hmm, first the “ras” file was in LIF format .. and now the lzma compress blocks are corrupted !!  
I googled this and tried to find a solution for this, **FOUND NOTHING** . How am I going to solve this ??   
One idea came in my mind .. “Strings” command and here is what I got :

[![strings](http://static.oschina.net/uploads/img/201401/19175936_vM18.jpg "strings")](http://static.oschina.net/uploads/img/201401/19175935_cqNB.jpg)

Aaaah ! so the blocks aren’t compressed with LZMA or anything ! and the whole “ras” firmware file is just big chunk of data in clear text.  
Ok, let’s try and find some useful STRINGS …

After some time searching “I” didn’t find the useful thing that will help us find vulnerabilities on the firmware !!

I didn’t give up …  
I just was thinking and questioning :

*   Me: What do you want from this firmware file !

*   Me: I want to find remote vulnerabilities that will help me extract the “admin” password.

*   Me: Does the web interface let you save the current configuration ? 

[![rom-save](http://static.oschina.net/uploads/img/201401/19175936_PIlE.jpg "rom-save")](http://static.oschina.net/uploads/img/201401/19175936_Jez1.jpg)

*   Me: yes !!

*   Me: Is the page password protected ? 

[![password-protected](http://static.oschina.net/uploads/img/201401/19175936_ypc5.jpg "password-protected")](http://static.oschina.net/uploads/img/201401/19175936_T61n.jpg)

*   Me: No !!! I tired to access that page on a different IP and it didn’t require a passowrd ! 

Ok, enough questions haha ..

Now, when I activated TamperData and clicked “ROMFILE SAVE”&#160; I’ve found out that the rom-0 file is located on “IP/rom-0″ and the directory isn’t password protected or anything.

So we are able to download the configuration file which contains the “admin” password. I took a look at rom-0 file and couldn’t figure out how to reverse-engineer it, and when you don’t know something it’s not a shame to ask for help .. and that’s what I did !  
I contacted “**Craig**” from [devttys0.com](http://devttys0.com/), he is an expect when it comes to hacking embdded devices . He’s a great guy and he replied to my email and pointed me to [http://50.57.229.26/zynos.php](http://50.57.229.26/zynos.php)which is a free rom-0 file decompressor .

[![decompress](http://static.oschina.net/uploads/img/201401/19175937_NJ9G.jpg "decompress")](http://static.oschina.net/uploads/img/201401/19175937_whRc.jpg)

When you upload and submit the rom-0 file there, the php page replies back with the configuration in clear text ( **INCLUDING THE PASSWORD** ) .

So what i need to do now is to automate the process of :

*   Download rom-0 file.

*   Upload it to [http://50.57.229.26/zynos.php](http://50.57.229.26/zynos.php)

*   get the repy back and extract the admin password from it.

*   loop this process to a range of ip addresses. 

And that’s exactly what I did, I opened an OLD OLD poc python script of mine that accessed routers via telnet using the default passwords. So what I just need to do now is to add some functionality to it.

Well I thought about&#160; this, and I’m posting this script online **ONLY FOR EDUCATIONAL PURPOSES.**

You can find the scripts here : [https://github.com/MrNasro/zynos-attacker/](https://github.com/MrNasro/zynos-attacker/)

Demo :

[![demo](http://static.oschina.net/uploads/img/201401/19175937_vATX.jpg "demo")](http://static.oschina.net/uploads/img/201401/19175937_xoJZ.jpg)

**PS : I OWN ALL THE IP RANGE I WAS SCANNING ” FOR SURE ![;)](http://static.oschina.net/uploads/img/201401/19180126_0Zrk.gif) “**

# Prevention :

Now ! how do you prevent attackers from downloading your rom-0 configuration file and manipulating your router ? This is pretty simple if you think about it ..  
You just have to forward port 80 on the router to and inused IP address on your network :   
[![forward](http://static.oschina.net/uploads/img/201401/19175938_gdlc.jpg "forward")](http://static.oschina.net/uploads/img/201401/19175937_XPrm.jpg)

**THATS ALL, **or if you want to play a little with attackers that are using scripts too .. just forward port 80 to you local http server and put a LARGE file in the root directory and name it rom-0 .. just let them download like 1GB rom-0 file haha haha .. I have also automated the process of port forwarding and I’m running the scripts daily just to prevent hackers from attacking weak users …

In the next post I’ll demonstrate how would a malicious hacker exploit this to hack TONS of networks and get a meterpreter/reverse_shell on every PC on the target network ..

Hope you enjoyed this analysis, if you have anything to add or any questions to ask don’t hesitate to contact me ! **BE THEIR HERO, HAPPY HACKING ![;)](http://static.oschina.net/uploads/img/201401/19180126_0Zrk.gif)**