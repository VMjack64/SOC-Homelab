# Introduction
To everyone who found this: Hello there, my name is Jack. I’d like to share with you something I’ve worked on for months: My own SOC analyst home lab, with every step of the process documented in a lengthy, comprehensive writeup.

Heavily inspired by MyDFIR’s 30-Day SOC Analyst Challenge series, the purpose for doing this project is to gain practical cybersecurity experience, particularly in the field of SOC analysis. When it comes to getting such experience, one thing was constantly mentioned on the internet: Home lab projects. So, I began seeking potential SOC analyst projects worth trying, and eventually stumbled upon the aforementioned YouTube series, which can be found here: https://www.youtube.com/playlist?list=PLG6KGSNK4PuBb0OjyDIdACZnb8AoNBeq6. This series proved fantastic for getting the hands-on experience I needed, as it has a perfect structure that makes it easy to follow along; each day has a specific learning goal in mind, and I get exposure to the various kinds of tools and events I’ll be expected to know as a SOC analyst. I made this writeup REALLY long not only because of my detail-oriented nature, but also because I incorporated a few twists of my own into the project, including a bonus challenge where I ran a piece of malware on the internet and analyzed its activity with Splunk and, to some extent, Wireshark. I’ll discuss these twists in more detail when they come up. Doing things this way certainly has prolonged the experience more than necessary, but ultimately proved worthwhile for the long-lasting knowledge. Without further ado, I’ll just jump into things right away.
## Table of Contents
Introduction

1. Conceptualizing the Lab
2. Splunk vs. ELK Stack
3. Beginning the Lab Setup
4. Creating the Windows Server
5. Prepare for Log Ingestion
6. Getting Logs Ingested Into Splunk
7. The First Step Towards Becoming a Proefficient Splunk User
8. Taking Things Seriously with Mythic
9. osTicket & Splunk
10. A Late Linux Addition
11. The RDP Investigation Part
12. What EDR to use?
13. The Bonus Challenge

Conclusion
