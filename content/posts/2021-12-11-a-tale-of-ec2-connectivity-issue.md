---
layout: post
title: A tale of EC2 connectivity issue
date: 2021-12-11 17:58:40.000000000 +00:00
 
---

  This happened 3 days ago. I received a message from one of our ML engineers that he can't access the EC2 server in the *us-east-1* region. I asked him about the error message and he said ssh is giving a time-out error.   
  
  
  So, I tried connecting to the server via EC2 connect feature  *(web shell)* that AWS provides, and even that said connection timed out.   
  
  
  Tried telnet to the endpoint and was the same also.   
  
  
  I thought maybe the server may be struck due to overload and restarted it. But the state was still the same once it came up. I saw the metrics of the server via Cloudwatch and everything was fine, saw system logs also and even that looked good.   

  Curiously opened [status.aws.amazon.com](https://status.aws.amazon.com)  and saw that **us-east-1** Region is having an outage.   

  Being a Reddit fan, opened **r/sysadmin** and I could see people all over the world complaining about AWS being down in that region and 1000's of memes on that topic. I told myself this could be mostly due to the AWS outage and I'll see once they fix it.   
  
  Cut to the next day because the outage took 19 long hours to fix the outage. long live  *SLA!*   
  
  I still was not able to connect to the instance post AWS fix. After digging for X time, turns out, the issue was with the subnet in which EC2 was launched. Someone mistakenly attached **NAT gateway** to the public subnet instead of the **Internet gateway**. Updated the correct config in the Route-table of the subnet and it worked.   
  
  One tiny missing detail can totally mess your mind up.   
  
  Another day of learning :D   
  
   Happy DevOpsing !!!   
  
