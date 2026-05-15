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

Something that I want to add is that as my first time attempting a SOC analyst project, good chances are I may have made some misjudgements along the way. Some I could think of include:

If you find anything interesting in the Splunk screenshots from the Part 13 analysis phase that I missed, by all means let me know; I can apply this knowledge and continue improving my skills in any future SIEM projects.

To cap off this writeup, I’d like to share my next steps going forward. I definitely want to work on the following, from highest priority to lowest:
- Learn AI. Having heard many stories about AI and the future, it’s become clear to me that if I’m going to succeed in today’s cybersecurity industry, having proficiency with this tool is a must. Knowledge of this tool would enable efficient SOC analysis workflows, using it to take care of repetitive alerts so that I can direct my focus onto more interesting ones.
- Get proficient with Wireshark. Because of my lackluster experience with Wireshark, I was not able to make good work of the packet capture for the view.exe executable like I hoped. Since MyDFIR said this is a tool that a SOC analyst should be familiar with, I’d want to make learning this as much of a priority as learning about AI. The site http://malware-traffic-analysis.net should be helpful in accomplishing this goal as soon as possible.
- Improve my SOC analysis workflow. While I’m learning about AI and Wireshark, I’d also like to find the time to improve my SOC analysis skills through additional malware home lab projects.
- Learn more about EDRs. This is not as urgent of a goal compared to the others, but still something worthwhile for the future. While I have dabbled with Aurora Lite in this lab, the tool only provides a limited experience with EDRs (which is a given, considering this is the free version of Aurora). Hopefully, I can get my hands on a fully-fledged EDR to fiddle around with for the full experience.

With that, it’s finally time for me to wrap this up. For those who took the time to read all of this from beginning to end, I want to extend my sincerest thanks to you all. I hope you came out of this with something that you can apply to your own projects.

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
