---
layout: post
title: How to change DNS server for Syrotech Router [BSNL FTTH]
date: 2021-04-27 18:26:11.000000000 +00:00
---
  
Ok, to be honest, I searched a lot on the internet to change ISP DNS servers to 3rd party servers (which you should !) for my router and couldn't find a direct article / steps to do that. Hence, this article.   
  
  
## Steps
  
  
- Open the router login page, which is mostly : [192.168.1.1](http://192.168.1.1) in your case.
- After logging in, navigate to Network page,  LAN IP Address tab.</li>
- Change the *Lan Dns Mode* to : **static**</li>
- Set the primary and secondary DNS address and click on Save/Apply.</li>
- Perform a reboot of router to apply the changes.</li>
  
 
![](/syro-dns.png)
 
  
  There are a lot of DNS providers out there most of them for free. However, please be wise while choosing them.   
  
  
  I have chosen <span style="text-decoration:underline;">1.1.1.1 </span>DNS as my primary server which is provided by Cloudflare.   
  
  
  I have set the secondary  server to <span style="text-decoration:underline;">8.8.8.8 </span> which is provided by Google so that if one of the service is down, it will fallback to another.   
  
  
  List of some of the best DNS providers list in [r/sysadmin ](https://www.reddit.com/r/sysadmin/comments/976aj2/updated_list_of_public_dns_resolvers_curated_by)

---

  Psst...... **Feeling Geeky ?**   
  
  
  Perform DNS benchmark tests :  https://www.grc.com/dns/benchmark.htm
  
  
  Cloudflare DNS validation test : [https://1.1.1.1/help](]https://1.1.1.1/help)
  
  
   Need more? Read the detailed guide for BSNL FTTH : ([Fiber optimization](https://www.reddit.com/r/bsnl/comments/ht37q4/guide_for_bsnl_ftth/)
  
  
  Worth reading about [3rd party DNS resolvers](https://www.howtogeek.com/664608/why-you-shouldnt-be-using-your-isps-default-dns-server/)
