---
title: Taking Down Chinese Hackers
tags: China Cyberwar LFI Data Exfiltration Phishing Spam Real-World
article_header:
  type: 
  image:
   #src: 
sidebar: 
  nav: blogs
---

*This is an ongoing effort to shut down people, evil people. I won't really call it a typical investigation, but a simple effort to hack someone evil.*

## Backstory

Yesterday I received a forwarded SMS from one of my friends, the SMS asked the receiver to verify their WhatsApp account in 24 hours and if the user fails to do so, their account would be shut. Being the hacker that I am, I noticed the sense of urgency and fear that the message tried to create with an attached phishing link. Hence, I decided to roll my sleeves up and dive in.

## Getting Hacked

*This section talks about what a successful exploitation looks like*

Ideally, when the user clicks on the link, the user lands on a phishing website, that informs the user about some illegal activities and how the user's account can be closed if they don't verify it.

![t](/Images/chinese/a.png){:.shadow}

Once the user clicks on verification, they are asked to enter their number and start the verification process.

![t](/Images/chinese/b.png){:.shadow}

After entering the number, a code is generated, which the website asks the user to enter in their WhatsApp mobile app.

![t](/Images/chinese/c.png){:.shadow}

Once the user enters this code in the `Link with phone number instead`, their WhatsApp gets linked with the attacker's device, meaning the attacker can now access the user's WhatsApp, just like you access your WhatsApp via `web.whatsapp.com`.

![t](/Images/chinese/d.png){:.shadow}

I tried to link my WhatsApp with a dummy number and as you can see, my number gets linked with Google Chrome browser running on a Windows system which the attacker is operating.

## Hacking Back

*This sections talks about what my exploitation looks like*

Once I landed on the phishing page, I quickly spun up BurpSuite and started scanning ports on the server. 

![t](/Images/chinese/1.png){:.shadow}

The phishing website was running on port 443, on port 80, it had an error message. Indicating, that the server is running aaPanel (a web hosted control panel for Linux servers).

![t](/Images/chinese/e.png){:.shadow}

I could not find the login or the admin panel, so I moved onto port 42201 which says not found, and then port 9999, where we have a web server running `ruoyi` (a permission management system).

![t](/Images/chinese/f.png){:.shadow}

The copyright mark says `2018-2023`, so I assumed the website has not been maintained since 2023(subjective), and I tried a few attack vectors, but couldn't find anything interesting. 

Now coming back to our phishing website on port 80, I started intercepting requests and browsing the website. While reviewing the burp sitemap, I stumbled upon an interesting end-point.

```bash
https://evil-website.com/api/upload
```

![t](/Images/chinese/2.png){:.shadow}

In the response, there's an error saying `Current request is not a multipart request`, which is obvious because it's not a POST request and I did not specify any file to upload. So what if I try to upload a file? 

![t](/Images/chinese/3.png){:.shadow}

What ! Really ?  did i just upload the file ? Can i access it too? 

To answer that, i started enumerating the right path in order to access the uploaded file, that's when i stumbled upon another interesting end-point.

```bash
https://evil-website.com/api/download
```

![t](/Images/chinese/4.png){:.shadow}

Okay, so there's an error saying `fileName` parameter is missing, what if i append the parameter name with the uploaded filename?

![t](/Images/chinese/5.png){:.shadow}

I can access the uploaded file. Obviously, the next thing that came to my mind was to get an RCE, but i can't do that, because this end-point simply downloads the specified file and does not view it.

So i tried what any other hacker would try, a local file inclusion attack. 

![t](/Images/chinese/6.png){:.shadow}

Voila ! A Local File Inclusion in a phishing website ? Very bad :(

## Juicing Out Data

Now that I had an LFI vulnerability, I started enumerating different files, including Configuration files, bash history, backup files, etc. The most interesting findings were.

- OS Information

![t](/Images/chinese/7.png){:.shadow}

- Bash History & aaPanel

![t](/Images/chinese/8.png){:.shadow}

We can see the user `azureuser` installed `aaPanel`, looking up the documents online, i was also able to enumerate the port and the path for the login panel of aaPanel.

![t](/Images/chinese/9.png){:.shadow}

### Dumping & Analysing

After a lot of enumeration, going through tonnes of log and configuration files, I was able to dump the `default` & `system` databases, which revealed the following information.

![t](/Images/chinese/10.png){:.shadow}

Starting with system db, it revealed system information including, top processes, network packets, CPU usage, load distribution etc.

![t](/Images/chinese/11.png){:.shadow}

Default DB revealed a lot more information including system path, usernames, password, access logs, error logs and most importantly it revealed how big the campaign is and all the phishing domains that have been set up by these hackers.

![t](/Images/chinese/12.png){:.shadow}

The following table revealed more system paths, performed operations, and the username.

![t](/Images/chinese/13.png){:.shadow}

### Stripping 'em Naked

Going through the entire database, I eventually got a path for database backup at `/www/backup/database/wsapp.sql`.

![t](/Images/chinese/14.png){:.shadow}

This gave me access to data of all the users who actually got phished and some other information as well.

![t](/Images/chinese/15.png){:.shadow}

The database dump also revealed that these hackers are not only targeting India but other countries as well including Spain. It also revealed information about native users of the application, including username, hash, salt, mobile number, IP, etc.

![t](/Images/chinese/16.png){:.shadow}

I was also able to enumerate plain text passwords for databases, SSH keys, IP addresses, etc, but that's beyond the scope, I've already spammed this blog with a lot of screenshots. I also know that I will become a good target after posting this blog. But that doesn't stop me from doing good. 

I achieved what I wanted to achieve by doing all this, but if someone is interested enough, this can also be taken up from a threat intel perspective and then tracing their location, etc.

I hope ya'll had a fun time reading this blog. Feel free to reach out for any questions.

