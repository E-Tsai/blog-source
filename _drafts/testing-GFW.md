# Testing GFW

<!--more-->

## IP Blacklist

Un-blacklisted website: baidu.com

![traceroute-baidu-cn](/home/elisa/Projects/blog-source/_drafts/testing-GFW/traceroute-baidu-cn.png)

Blacklisted website: google.com

![traceroute-google-cn](/home/elisa/Projects/blog-source/_drafts/testing-GFW/traceroute-google-cn.png)

last hop: 202.97.36.221

![ip-filtering-device](/home/elisa/Projects/blog-source/_drafts/testing-GFW/ip-filtering-device.png)

A Cisco device.



## DNS Hijacking



![dn-hijacking](/home/elisa/Projects/blog-source/_drafts/testing-GFW/dn-hijacking.png)



## Content Censorship

Innocent TCP:

![hello-tcp](/home/elisa/Projects/blog-source/_drafts/testing-GFW/hello-tcp.png)

Innocent HTTP:

![http-hello](/home/elisa/Projects/blog-source/_drafts/testing-GFW/http-hello.png)

Sensitive words in HTTP:

![http-freegate](/home/elisa/Projects/blog-source/_drafts/testing-GFW/http-freegate.png)

TCP resets appear.

Comparing server side and client side:

![freegate-server-vs-client](/home/elisa/Projects/blog-source/_drafts/testing-GFW/freegate-server-vs-client.png)

Another one: 

![successful-tcp-reset](/home/elisa/Projects/blog-source/_drafts/testing-GFW/successful-tcp-reset.png)

what is TCP reset attack:

