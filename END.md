Previous Part                                   | Return to Introduction                 
----------------------------------------------- | ---------------------------------------
[Part 13: The Bonus Challenge](/PART13BONUS.md) | [Introduction](/README.md#introduction)

# Conclusion / Closing Thoughts
Whew, this was quite the writeup.

This personal SOC analyst project truly was an experience to behold; the unique set of challenges I've faced as a consequence of my setup really forced me to sit back and comprehend the situation much of the time. Many of those challenges, such as the timesink from going through the `view.exe` event logs, proved difficult to accomplish, and a few were even impossible for me. But for the most part, I persevered, and in turn, gained a fundamental understanding of the following:
- Networking (ports, IP addresses, subnets, firewalls, access control lists (ACLs), classless inter-domain routing blocks (CIDR blocks, or subnet masks))
- Detection engineering with EDRs (Endpoint Detection & Responses)
- Using Splunk (& SIEM tools) for event log analysis of suspicious login activity and malware
- Troubleshooting

In doing this home lab, I also got a couple of takeaways:
- For networking, it's recommended to draft a diagram of the setup. It really does help with visualization.
- Whether it's using a different SIEM tool from the one specified, or a different CIDR block notation, even small deviations can be very effective in grasping new concepts compared to following step-by-step (if applicable).

Something I want to add is that as my very first time attempting a SOC analyst project, good chances are I made some misjudgements along the way. Some examples that come to mind include:
- Not having my deployment server & search head functionalities in the same Ubuntu instance, with the other Ubuntu instance acting as a heavy forwarder (indexer). This would've made the Linux log ingestion process & analysis straightforward.
- Not using my malware dashboard for the bonus challenge.

There's probably more out there, especially in the real malware analysis part. If you find anything or have some feedback to offer, by all means let me know. I can apply this knowledge and continue improving my skills in future SOC analyst projects.

To cap off this writeup, I’d like to share my next steps going forward. I plan to do the following, from highest priority to lowest:
- **Learn AI**. I've heard so many stories about AI that at this point, it's abundantly clear to me that having proficiency with this tool should be my highest priority. As such, this will be my next goal. Knowledge of this tool would enable efficient SOC analysis workflows by taking care of repetitive tasks, letting me direct my focus onto more interesting ones.
- **Expand knowledge on networks**. While this lab helped me learn the fundamentals of networking, I want to continue expanding this knowledge, getting a grasp on things like the TCP/IP model, OSI model, and TLS.
- **Get proficient with Wireshark**. Because of my lackluster experience with Wireshark, I was not able to make good work of the packet capture for `view.exe` like I hoped. [MyDFIR said this is a tool that a SOC analyst should be familiar with](https://www.youtube.com/shorts/BzRXa4IGOy8), so I want to get a grasp on it as soon as possible. I plan on taking advantage of [this site](http://malware-traffic-analysis.net) to help accomplish this goal with ease.
- **Learn more about cloud service providers (AWS, Microsoft Azure, etc.)**. I'd also like to get a handle on cloud stuff as soon as possible so that I have a better understanding on how to monitor them for suspicious/malicious activity as a SOC analyst.
- **Learn more about EDRs**. I don't consider this as urgent of a goal compared to the others, but it's still something worthwhile for the future. For the full experience, I'd like to get my hands on a fully-fledged EDR to fiddle around with, if possible. LimaCharlie is one that I'm eyeing in this regard.

As I continue accomplishing more goals on my roadmap, I'm betting on my SOC analysis workflow improving in the process and therefore me becoming a better SOC analyst.

With that, it’s finally time for me to wrap things up. For those who took the time to read everything from beginning to end, I want to extend my deepest gratitudes to you all. I hope you came out of this with something that you can apply to your own projects.

Until then, keep on Splunking.  
Jack

## Additional Sources
- Pipes
  - https://www.splunk.com/en_us/blog/security/named-pipe-threats.html
  - https://versprite.com/vs-labs/microsoft-windows-pipes-intro/
- CIDR Blocks
  - https://wiki.teltonika-networks.com/view/What_is_a_Netmask%3F
  - https://dnsmadeeasy.com/resources/subnet-mask-cheat-sheet

Previous Part                                   | Return to Introduction                 
----------------------------------------------- | ---------------------------------------
[Part 13: The Bonus Challenge](/PART13BONUS.md) | [Introduction](/README.md#introduction)
