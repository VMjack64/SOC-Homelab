# Part 1: Conceptualizing the Lab (Day 1)
Before starting things off, I first needed to conceptualize what my home lab would look like. For this, I watched Day 1 of MyDFIR’s (Steven's) series to get a good idea of what to expect. Steven’s lab for the series consists of 6 cloud servers (created using the VULTR cloud provider), 1 virtual machine (VM; Kali Linux), and his laptop for connecting to the servers. The 6 servers are:
- Mythic C2 (Command and control) server
- Elastic & Kibana server (using the ELK stack)
- Fleet server
- Windows server w/ RDP enabled
- Ubuntu server w/ SSH enabled
- osTicket server (for Ticketing)

When thinking about why he created each server, a good rule of thumb popped up in my mind: 
> Before building a home lab, first think about the learning goals that you hope to achieve from doing it.

Some examples include:
- Learn to use Splunk efficiently
- Learn to create firewall rules to block traffic

When reviewing your learning goals, it’s preferable to be as specific as possible. So instead of “Learn to use Splunk efficiently”, something like the following is a better motive:
- Learn to route traffic to IDS/IPS, then to Splunk

and/or
- Learn to generate telemetry for various types of logs and analyze them

Thinking about these goals will provide a good idea as to what instances/components will be needed for the lab. For me, the last 3 learning goals mentioned above just so happen to be the ones that I hope to achieve from doing this lab. I want to get the most out of my learning experience; knowing myself, I knew I wanted to do things differently. So, I put my own spin on Steven’s series - instead of using the ELK stack & VULTR, I decided to use Splunk and Amazon Web Services (AWS) as my SIEM tool and cloud service provider, respectively.

With that said, I must admit that my original setup was an imitation of Steven’s setup, but with the Mythic C2 & SSH Ubuntu servers excluded. I had done this because of redundancy & for the sake of time. But like some spiritual being watching over with a sense of disappointment, reminders of my learning goals in one form or another led me to include those servers. Coupled with the unique challenges posed from using Splunk and AWS, I made changes to my setup throughout the project, and eventually settled upon this: 

