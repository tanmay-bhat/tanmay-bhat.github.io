---
layout: post
title: How to install netstat tool in openSUSE 15
date: 2021-01-25 01:03:12.000000000 +00:00
 
background: /terminal.jpg
permalink: "/2021/01/25/how-to-install-netstat-tool-in-opensuse-15/"
---
  Even though the **netstat **tool is depreciated, sometimes we can't stop the old habit and we arrive at a situation where its difficult to adapt to new things.   
  
  
  Actually we should be using **ss **tool installed of netstat !   
  
  
  All common network related tools are bundled with package net-tools.   
  
```
nwlab:/etc # rpm -qa | grep net-tools
net-tools-2.0+git20170221.479bb4a-lp152.5.5.x86_64
```  
  
  However in openSUSE 15, the team decided to knock it off from net tools package!   
  
  
  <span style="text-decoration:underline;">**So, the solution ?**</span>   
  
  <!-- wp:paragraph {"align":"justify"} -->   
<p class="has-text-align-justify">Install the package net-tools-deprecated by typing:   
  
  
  ```sudo zypper install net-tools-deprecated```   
  
```console
nwlab:/etc # rpm -qa | grep net-tools
net-tools-deprecated-2.0+git20170221.479bb4a-lp152.5.5.x86_64
```
  
  Once installed, netstat should work totally fine now !   
  
  
```console
nwlab:/etc # netstat -ano | grep 9000
tcp6 0 0 :::9000 :::* LISTEN off (0.00/0/0)
```
  
