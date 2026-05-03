# Part 11: The RDP Investigation Part (Day 27)
With my investigation of the SSH authentication activity wrapped up, it’s time for the Windows Server to have its activity investigated. Unlike the case with Linux, no premature troubleshooting shenanigans had to be done here; I can just jump straight into investigating the RDP activity. Another thing of note is that the logs I have generated are similar to the kinds of logs Steven had in the Day 27 video, since authenticating into my Windows instances in AWS is pretty much the same as authenticating into Windows instances in VULTR, just with an extra step in between. Putting that in mind, I’ve excluded Question 4 from the previous investigation.
Asides the one exclusion, I’m using the same questions as before for this investigation, since I’m still dealing with brute force activity, albeit with a different protocol involved. Given the ease in workload for this part compared to the previous investigation, I figured I’d give myself a bit more work by generating a new set of brute force events that I’ll investigate, on top of my old set.
Question 1: Any successful attempts (not from my IP address)? If so, what did the attacker do?
A quick glance at my dashboard shows some successful authentications via RDP, but all of them came from my IP:

To verify further, I ran a quick search of the “windows-server-events” index for events with event ID 4624. Checking the Source_Network_Address field, I see a blank value and my IP address making up the available values for the field:


Filtering down to the events with the blank value, I checked the Logon_Type field for any 3s, 7s, and/or 10s. None of them appeared, meaning these events were spawned by critical Windows processes, thus no successful attempts from any outsiders occurred here.
Question 2: Is the IP address known to perform brute force attacks? And what usernames did the suspicious IP address target?
Like last time, I’m only going to focus on the country with the most events. I say country in particular because I saw a LOT of events coming from the Netherlands:

Because of the significant amount of events coming from the same city, alongside the three IPs sharing the same CIDR block (the first three digits are the exact same, with only the last digit differing), I was led to believe that this might be a malicious organization executing a distributed brute forcing offensive here. Therefore, for this situation, I’ve decided to focus on the three IPs shown.
First off, I checked the event count for each IP. The IP in the middle (185.156.73.173) didn’t generate much events:

The top & bottom IPs (185.156.73.169 & 185.156.73.69, respectively), on the other hand, comprised the bulk of the events logged:


Next, I ran the IPs through AbuseIPDB:
Top IP



Middle IP



Bottom IP



Then, I ran the IPs through GreyNoise:
Top IP

Middle IP

Bottom IP

GreyNoise didn’t observe any activity for either of the three IPs. However, I only realized when writing up my draft for this part that the time window only went back as far as one day. Noticing this, I reran one of the IPs through GreyNoise to try and see if I could find a way to view the activity from the past 180 days (~6 months). A little bit of fiddling later, I quickly realized that this is impossible in my situation; GreyNoise restricts the time window to 1-2 days maximum for free plan accounts, which is what I’ve been using all this time. So, I’m relying solely on AbuseIPDB for this.
Because of this restriction, I ultimately decided to look for an alternative to GreyNoise on the internet. This led me to CrowdSec, which happens to be open source. Running the bottom IP into the tool, CrowdSec provides some additional information about the IP, to a similar level of detail that GreyNoise does:



Note that all the CrowdSec screenshots were taken this February, a few months after my first lookup session of these IP addresses. Despite a differing location here compared to what was in the screenshots from my initial investigation, CrowdSec still lists the exact same AS (Autonomous System) name from GreyNoise, so it’s still the same IP. The activity time window only goes back as far as November, but even within that time frame, there have been many reports about malicious brute forcing activity coming from the IP, meaning it’s still as active as before. Topping things off, CrowdSec even shows a graph of what countries were targeted most by this IP, which in this case is the US.
Satisfied with CrowdSec’s results, I proceeded to run the other two IPs through it as well:
Middle IP



Top IP



The CrowdSec reports state that the three IPs are all in the same CIDR block, with the same AS name. This likely indicates that the IPs make up a botnet that executes automated mass brute force attacks against target systems, as well as scanning them for vulnerabilities to exploit to gain access. However, I wanted more information to further verify this theory, so I went and queried the AS name:



Although this is only two handfuls of results returned, the information is enough to confirm my suspicions; this is indeed a botnet. Additionally, the “Top Behaviors” column on the left reveals that the botnet targets the HTTP and TCP protocols for its scans. When I was filtering for results involving these scanning behaviors, I uncovered even more behavioral information about the botnet, which included things like DDoS attacking and exploitation of vulnerabilities in the CVE list, such as:
SAP NetWeaver - RCE (CVE-2025-31324)
PAN-OS - RCE (CVE-2024-3400) and exploitation of vulnerabilities in the CVE list to deploy malicious commands and software
These CVE vulnerabilities only target specific software and infrastructure. With all this new information, it reinforces the fact that this is an active malicious botnet scanner capable of hacking. This sentiment is also somewhat echoed in the AbuseIPDB reports:
Top IP

Middle IP

Bottom IP

While there’s probably even more information about the botnet I haven’t uncovered, I feel like I’ve reached a satisfactory enough conclusion with the information I know about, so I won’t continue digging deeper beyond this point.
With the IP addresses now determined to be part of a malicious botnet, there’s still the other part of the question to be answered: What usernames did it try using? Well, a LOT:

The screenshot only captures the tail end of the list, but looking at the scroll bar on the right should be a sign as to how many usernames this botnet used. With this many usernames involved, determining how many attempts used a unique username (and consequently, determining the number of unique usernames) would be incredibly tedious. Fortunately, I can always run a query to determine this outcome:

Ultimately though, for this part of the question, I just screenshotted the entire list of usernames used by the botnet and pasted them into my osTicket report.
Question 3: What activity did I perform in one of my successful authentication sessions?
As my dashboard doesn’t include the logs for successful RDP activity, I used the knowledge on Event IDs & Logon Types from Part 7 and I ran the following query to view all of them:

Back in the SSH investigation, I utilized my first authenticated session to answer this question. This time, to mix things up, I’m using a successful authentication session somewhere in between the first and last event. I’ve chosen to settle with the following event created on September 3, 2025, 4:20:28.000AM UTC:

Referring to my correlation technique for the text file back in Part 7, I used that knowledge and grabbed the logon ID for the event:

Then, I ran another query where I searched the “windows-server-events” index for this ID:

Immediately, I spotted the disconnect time for the session, which was 9:08:07.000AM UTC on the same day. Other than that, the rest of the logs didn’t unveil anything significant, so I changed the query’s index over to the Sysmon one. There, I uncovered three process created events:
Two of those events involve the notepad.exe process. In both cases, a text file named “hello-world.txt” was opened. The events happened a few hours apart from each other. However, the directory location differed between the two:


The third event involved the DismHost.exe process (Dism Host Servicing Process). This process was created by the cleanmgr.exe executable, which removes any unnecessary files from the machine:


Given the ParentCommandLine, this process runs automatically against the instance’s hard disk. And looking at the CommandLine & Image fields, this event targeted a file in the temp directory, or temporary directory, which as the name states, houses files currently in use. Judging by the timestamp, which happened about an hour after the first instance of me opening the “hello-world.txt” file, this event likely occurred as a result of closing the Notepad app, which would’ve freed up space used for rendering the text.
With all this information, I could draw the surface-level conclusion that I simply opened up a text file via Notepad, closed it, then opened another text file with the same name via Notepad. Though, on a deeper level, the change in directories between the two notepad.exe events might be indicative of stuff happening behind-the-scenes. indicate a few things. Namely, either I did in fact have two files with the same name under different directories, or I created the second text file shortly after the first event occurred. Now, this is all answered in Part 7 of this lab; the timestamps do match the time when I did my preemptive warmup for getting familiar with Splunk. I bring this point up as a mental note to keep in mind something to take note of when doing real-world analysis work.
osTicket Report
With all questions of this investigation answered, I submitted my report to the RDP ticket in osTicket, which includes a text file containing all the usernames used by the 3 IP addresses in Question 2:


