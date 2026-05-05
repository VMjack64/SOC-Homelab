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

In need of another source instead of solely relying on AbuseIPDB, I sought an alternative to GreyNoise. This led me to [CrowdSec](https://www.crowdsec.net/search), which just so happens to be open source. Running the bottom IP into the tool, CrowdSec does provide some additional information about the IP, to a level of detail that's favorably comparable to GreyNoise:
![](/screenshots/261.png)
![](/screenshots/262.png)
![](/screenshots/263.png)

Due to conducting this lookup, as well as all other CrowdSec lookups, in February, the timeframe only went back as far as November. Despite that, there have been many reports about malicious brute forcing activity coming from the IP, meaning it’s still as active as before. The location specified in CrowdSec differs from my initial GreyNoise lookup, but the same AS (Autonomous System) name is specified in both, reaffirming that it's still the same culprit:
- GreyNoise
  ![](/screenshots/264.png)
- CrowdSec
  ![](/screenshots/265.png)

Satisfied with CrowdSec’s results, I proceeded to run the other two IPs through it as well:
- Middle IP
  ![](/screenshots/266.png)
  ![](/screenshots/267.png)
  ![](/screenshots/268.png)
- Top IP
  ![](/screenshots/269.png)
  ![](/screenshots/270.png)
  ![](/screenshots/271.png)

Based on the CrowdSec reports for the IPs, it is likely that the three machines make up part of a botnet whose purpose entails executing automated scans for exposed machines to exploit, and carrying out automated brute force attacks against these targets to gain access. However, I wanted more information for further verification, so I went and queried the AS name:
![](/screenshots/272.png)
![](/screenshots/273.png)
![](/screenshots/274.png)

Although this is only two handfuls of results returned, all the information provided is enough to solidify my suspicions that this is indeed a brute forcing botnet. Furthermore, under the **Top Behaviors** column on the left, there's a decent amount of reports stating that the botnet targets the HTTP and/or TCP protocols for its scans, indicating that the targets are typically web applications and/or remote machines, respectively. I looked into the results involving these scanning behaviors, and ended up uncovering even more behavioral information about the botnet, which included things like DDoS attacking and exploitation of vulnerabilities in the CVE list, such as:
- SAP NetWeaver - RCE ([CVE-2025-31324](https://tracker.crowdsec.net/cves/CVE-2025-31324))
- PAN-OS - RCE ([CVE-2024-3400](https://tracker.crowdsec.net/cves/CVE-2024-3400))

This sentiment is also somewhat echoed in the AbuseIPDB reports, further driving home the malicious nature of this botnet:
- Top IP
![](/screenshots/275.png)
- Middle IP
![](/screenshots/276.png)
- Bottom IP
![](/screenshots/277.png)

While there’s probably even more information about the botnet I don't know about, I feel like I’ve reached a satisfactory conclusion with everything I've uncovered, so I won't continue digging any deeper.

Now that I know the IP addresses are associated with a malicious botnet, there’s still the other part of the question: What usernames were targeted? A quick glance at the RDP failed authentications on my dashboard reveals a lot:
![](/screenshots/278.png)

To determine the exact count of unique usernames here quickly, I ran the following query:
![](/screenshots/279.png)

Because of the large amount of unique usernames, I've only shown the last few entries in the list, with the rest pasted into my osTicket report. Considering the kinds of usernames used, the botnet probably carries out credential stuffing attacks, a kind of brute forcing that utilizes username and password credentials from previous breaches.

## Question 3: What activity did I perform in one of my successful authentication sessions?
As my dashboard doesn’t include the logs for successful RDP activity, I ran the following query to view all of them:
![](/screenshots/280.png)

For this question, I’ve settled upon this event:
![](/screenshots/281.png)

Utilizing the correlation technique I learned back in [Part 7](/PART7.md#correlating-text-file-creation--successful-authentication-activity), I grabbed the logon ID for the event:
![](/screenshots/282.png)

Then, queried the `windows-server-events` index for all events containing this ID:
![](/screenshots/283.png)

Asides getting the disconnection time for the session, which was 9\:08\:07.000AM UTC on the same day, nothing useful came up in this search, so I modified the query, changing the index to the Sysmon one. There, it's revealed that three process created events occurred throughout the session:
- Two of those events involve the `notepad.exe` process. In both cases, a text file named `hello-world.txt` was opened. The events happened a few hours apart from each other. However, the directory location differed between the two:
![](/screenshots/284.png)
![](/screenshots/285.png)
- The third event involved the `DismHost.exe` process (Dism Host Servicing Process). This process was created by the `cleanmgr.exe` executable, which removes any unnecessary files from the machine:
![](/screenshots/286.png)
![](/screenshots/287.png)
Given the ParentCommandLine, this process runs automatically against the instance’s hard disk. And looking at the CommandLine & Image fields, this event targeted a file in the temp directory, or temporary directory, which as the name states, houses files currently in use. Judging by the timestamp, which happened about an hour after the first instance of me opening the “hello-world.txt” file, this event likely occurred as a result of closing the Notepad app, which would’ve freed up space used for rendering the text.

With all this information, I could draw the surface-level conclusion that I simply opened up a text file via Notepad, closed it, then opened another text file with the same name via Notepad. Though, on a deeper level, the change in directories between the two notepad.exe events might be indicative of stuff happening behind-the-scenes. indicate a few things. Namely, either I did in fact have two files with the same name under different directories, or I created the second text file shortly after the first event occurred. Now, this is all answered in Part 7 of this lab; the timestamps do match the time when I did my preemptive warmup for getting familiar with Splunk. I bring this point up as a mental note to keep in mind something to take note of when doing real-world analysis work.

## osTicket Report
With all questions of this investigation answered, I submitted my report to the RDP ticket in osTicket, which includes a text file containing all the usernames used by the 3 IP addresses in Question 2:


