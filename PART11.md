# Part 11: The RDP Investigation Part (Day 27)
With my investigation of the SSH authentication activity wrapped up, next up is the Windows server's RDP activity. Thankfully, since Splunk naturally parses the Windows events perfectly fine, I can just jump straight into the investigation. I'm using the same questions from the previous part here, except Question 4, since there isn't any variety for me to work with, in terms of what types of failed authentication events for Windows exist. To try and make this investigation somewhat interesting, I've exposed the Windows server to the internet for one more telemetry gathering session, which I'll analyze alongside the old telemetry.

## Question 1: Any successful attempts (not from my IP address)? If so, what did the attacker do?
A quick glance at the RDP successful authentications on my dashboard shows some successes, but all of them came from my IP:
![](/screenshots/245.png)

To verify further, I searched the `windows-server-events` index for events with event ID 4624. In the `Source_Network_Address` field, I see a blank value and my IP address comprising the available values:
![](/screenshots/246.png)
![](/screenshots/247.png)

Filtering for the events with the blank value, I checked the `Logon_Type` field for a 3, 7, and/or 10. Neither of them appeared, meaning these events were spawned by critical Windows processes. Thus, no successful attempts from any unknown entity occurred here.

## Question 2: Is the IP address known to perform brute force attacks? And what usernames did the suspicious IP address target?
Like last time, I’m only going to focus on the country with the most events. There were a LOT coming from the Netherlands:
![](/screenshots/248.png)

The significant amount of events, plus the similarities between the three IPs, led me to suspect a group operation at play here, executing a distributed brute forcing offensive. Therefore, for this situation, I want to focus on the three IPs shown.

First off, I checked the event count for each IP. The middle IP (185.156.73.173) didn’t generate much events:
![](/screenshots/249.png)

While the top & bottom IPs (185.156.73.169 and 185.156.73.69, respectively) comprised the bulk of the events logged:
![](/screenshots/250.png)
![](/screenshots/251.png)

Afterwards, I ran the IPs through AbuseIPDB:
- Top IP
  ![](/screenshots/252.png)
  ![](/screenshots/253.png)
  ![](/screenshots/254.png)

- Middle IP
  ![](/screenshots/255.png)
  ![](/screenshots/256.png)
  ![](/screenshots/257.png)

- Bottom IP
  ![](/screenshots/258.png)
  ![](/screenshots/259.png)
  ![](/screenshots/260.png)

Then, I ran the IPs through GreyNoise. GreyNoise didn’t observe any activity from either of them. However, while writing my first draft for this part, I reviewed the screenshots I took and realized that the timeframe for all three only went back as far as one day. Rerunning one of the IPs, I tried to find a way to expand the timeframe to go back even further, but was quickly hit with the revelation that this is impossible in my situation; GreyNoise restricts the timeframe to 1-2 days maximum on free plan accounts, which is what I’ve been using all this time.

In need of another source instead of solely relying on AbuseIPDB, I sought an alternative to GreyNoise. This led me to [CrowdSec](https://www.crowdsec.net/search), which just so happens to be open source. Running the bottom IP into the tool, CrowdSec does provide some additional information about the IP, to a level of detail that's comparable to GreyNoise:
![](/screenshots/261.png)
![](/screenshots/262.png)
![](/screenshots/263.png)

Due to conducting this CrowdSec lookup (plus all other CrowdSec lookups) in February, the timeframe only went back as far as November. Despite that, there have been many reports about malicious brute forcing activity coming from the IP, meaning it’s still as active as before. The location specified in CrowdSec differed from my initial GreyNoise lookup, but the exact same AS (Autonomous System) name is specified in both, meaning it's still the same:
- GreyNoise
  ![](/screenshots/264.png)
- CrowdSec
  ![](/screenshots/265.png)

Note that all the CrowdSec screenshots were taken this February, a few months after my first lookup session of these IP addresses. Despite a differing location here compared to what was in the screenshots from my initial investigation, CrowdSec still lists the exact same AS (Autonomous System) name from GreyNoise, so it’s still the same IP. The activity time window only goes back as far as November, but even within that time frame,  Topping things off, CrowdSec even shows a graph of what countries were targeted most by this IP, which in this case is the US.
Satisfied with CrowdSec’s results, I proceeded to run the other two IPs through it as well:
Middle IP



Top IP



The CrowdSec reports state that the three IPs are all in the same CIDR block, with the same AS name. This likely indicates that the IPs make up a botnet that executes automated mass brute force attacks against target systems, as well as scanning them for vulnerabilities to exploit to gain access (which, sure enough, such a thing exists). However, I wanted more information to further verify this theory, so I went and queried the AS name:



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

## Question 3: What activity did I perform in one of my successful authentication sessions?
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


