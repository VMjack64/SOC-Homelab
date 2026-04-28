# Part 8: Taking Things Seriously with Mythic (Days 18, 19, 20, 21, 22, 28)
Having spent a good amount of time searching logs & creating dashboard panels with Splunk, I definitely have gotten to the point where I have a grasp on the software fundamentals. With that, I took things to the next step and began analyzing some malware. Specifically, I’m going to run an Apollo executable on the Windows instance and try to analyze its activity with Splunk.

## Attack Preparations
Before running the executable, I need to do some preparation work. The first task on the list is to construct a brute force attack diagram outlining how the attack is going to play out:
![](/screenshots/94.png)
![](/screenshots/95.png)
![](/screenshots/96.png)

Next, I need to create the instance that I will be installing Mythic in. But first, I need to create the subnet that will house the instance. I gave this subnet the following specifications:
![](/screenshots/97.png)
![](/screenshots/98.png)
![](/screenshots/99.png)
![](/screenshots/100.png)
![](/screenshots/101.png)

After creating the subnet, I built the instance, giving it the following:
  - Name: MYDFIR-Mythic
  - AMI: Ubuntu, Ubuntu Server 24.04 LTS
  - Architecture: 64-bit (x86)
  - Instance type: c7i-flex.large. Recommended to run Mythic on a machine with 2 CPUs & 4GB of RAM, which this option has exactly.
  - Key pair: RSA type, .pem format
  - Network settings:
    - VPC: MYDFIR-30Day-SOC-Challenge
	- Subnet: mythic-subnet
	- Auto-assign public IP: Enabled
	- Firewall (security groups): Create security group, with the following settings:
	  ![](/screenshots/102.png)
	  ![](/screenshots/103.png)
  - Configure storage: 1 volume only:
	- Text box: 30 GiB
	- Dropdown box: Left as whatever option is selected

Lastly, I want a Kali Linux VM (virtual machine) for operating Mythic safely. Conveniently, I have one in VirtualBox ready to go from doing the Basic Home Lab series mentioned previously. In terms of internet access, for the best case scenario, I want the Kali VM connection set to NAT (Network Address Translation); [Part 2](https://www.youtube.com/watch?v=5iafC6vj7kM&pp=ygUVbXlkZmlyIGJhc2ljIGhvbWUgbGFi) of the Basic Home Lab series provides concise summaries on the various types of VM network connections.

## Attack Simulation
> [!CAUTION]
> Only perform these attacks on machines that YOU own, otherwise you could face some legal ramifications.

With all the necessary preparations done, I began the process of simulating the attack in the diagram from the previous section. After connecting to the Mythic instance via SSH & updating its repositories, I was able to install Mythic onto the instance by following the Day 20 video without any caveats. Then, I followed along with the Day 21 video to execute the attack, but with a couple of alterations:
1. I’ve opted to stick with the default Windows server password provided upon instance creation time instead of changing it, since I thought AWS will automatically roll back the changes.
2. I’ll be using the `hello-world.txt` file I created back in the Log Analysis Warmup section, instead of creating a `passwords.txt` file.

In trying to follow along with the video, I ran into a problem with Crowbar. When running the tool, I get the following error:

`xfreerdp: /usr/bin/xfreerdp path doesn't exists on the system`

A straightforward issue at first glance, I went and tried to install xfreerdp onto Kali, but was having a difficult time finding the package. I eventually found out that the package is obsolete and replaced by xfreerdp3 in the Kali repositories. Yet, Crowbar still needed the old xfreerdp executable for RDP to work (at the time of writing this). To fix this conundrum, I installed the xfreerdp3 package, then created a symlink to the executable named `xfreerdp` and moved the symlink file to the `/usr/bin/` directory.

Afterwards, I reran the Crowbar command, and this time it successfully executed. Unfortunately, I ran into another problem: Crowbar was not able to find the password in the file. I double checked a few things:
  - The file to confirm it’s saved with the Windows instance password already inputted.
  - The terminal to make sure I’m running the command in the same directory location as the file.
  - The public IP address of the Windows instance on AWS to make sure it’s the correct one.
  - The username on AWS (in this case, ‘Administrator’) to make sure it’s correct.

Even after verifying all of these, Crowbar still couldn’t find the password after running the command again. When I tried debugging the command, I noticed that Crowbar seemed to get stuck in a loop. So at that point, I just abandoned Crowbar and checked out other similar tools. Eventually, I tried out Hydra, which was mentioned in the Day 11 video, and ran the following command in the same directory as the `.txt` file specified:

`hydra -l Administrator -P mydfir-wordlist.txt rdp://[<public-ip-of-windows-server>/32]:3389`

When running the command the first time around, Hydra didn’t find the password, just like with Crowbar. However, I immediately ran it again (as I assumed the failure was due to improper asynchronization), and this time it actually worked:
![](/screenshots/104.png)

With the “brute force” attack successful, I connected to the instance from Kali via xfreerdp(3) to complete the first phase.

> [!NOTE]
> When specifying the password in the `xfreerdp` command, it’s recommended to enclose it in parentheses, since the password might contain special characters that can mess with the command formatting.

After connecting to the Windows server, I executed the next couple of phases of the attack. First, I performed host & network discovery by referring to [this site](https://car.mitre.org/analytics/CAR-2016-03-001/) for the types of discovery commands that attackers typically use. Then, after disabling Windows Defender, I navigated to the Mythic web GUI in Kali to build the payload that will be implanted into the server.

After generating the payload, I ran the `wget` command in the Mythic SSH terminal to grab the payload, making sure to replace the IP address in the link with the private IP address of the Mythic instance. I repeated this replacement scheme with the `Invoke-WebRequest` command in Windows, since I have the attacker and target instances in the same VPC. After downloading & running the payload, I checked its status and saw a `SYN_SENT` reading, indicating the connection wasn’t established. Right away, I saw the problem:
![](/screenshots/105.png)

During the payload building process, I specified the callback host as the localhost IP address (127.0.0.1), which is a loopback IP address to the machine itself. So in this context, the payload is trying to connect to a Mythic C2 server on the Windows instance, not the one on the Mythic instance. Therefore, I had to repeat the process of generating the payload, this time using the private IP address of the Mythic instance as the callback host. When trying to download the new payload onto the Windows instance, I got the following error:
![](/screenshots/106.png)

Considering I downloaded the old payload perfectly fine, it didn't take me long to figure out the problem: I needed to run the Mythic instance's Python server in the same directory as the payload.

After downloading & running the new payload, I checked its status and saw an `ESTABLISHED` reading, indicating the connection was successful. I didn’t even have to specify `ufw allow 80` on the Mythic SSH terminal.

With the C2 established, I ran a couple of discovery commands, then tried to download the `hello-world.txt` file. Despite specifying the absolute path to the file, Mythic was somehow having a hard time finding it. After a few more failed attempts, I tried a detour using `ls`, and through it, I was finally able to get the file:
![](/screenshots/107.png)
![](/screenshots/108.png)
![](/screenshots/109.png)

## Investigating Mythic C2 Activity in Splunk
With the simulated attack carried out, I viewed the telemetry generated in Splunk to begin analyzing it. For this analysis, I wanted it to be as close to a real-life scenario as possible; this assumes absolutely no knowledge on things like what malicious activity was performed, the malicious process(es) involved, or the method used to access the Windows instance. To facilitate this, I assumed the role of a L1 SOC analyst for a company. I boot up my computer & see the following message:

_“There has been reported suspicious activity on the Windows server around September 10. Please look into it.”_

At first, I had no clue where to start, so I checked out a few videos for some ideas, including the Days 22 & 28 videos, plus [this one](https://youtu.be/OAuVYbn1m3A?t=1474). The last video provided a good tip for starting out: Look for network connection events (Sysmon event ID 3), as malware usually makes a network connection of some sort to function, such as interacting with a C2 server to execute attacks remotely. Doing so for the date of September 10, the first thing I noticed is unusually high activity around the 4AM mark on the timeline:
![](/screenshots/110.png)

Clicking into that timeframe, I looked at the fields present under **Interesting Fields**, starting from the smallest number of unique values to the largest. When I checked the `dest_port` field, I immediately spotted a large number of events involving port 3389, an indicator that a brute force attack via RDP occurred:
![](/screenshots/111.png)

I switched over to the `windows-server-events` index, checking the Windows event IDs for a successful authentication attempt, and found 5 successes, sparking some concern:
![](/screenshots/112.png)

Filtering for these events, I looked at the `Logon_Type` field, and saw a logon type 3, which makes the situation highly alarming as it indicates that the brute force attack was successful:
![](/screenshots/113.png)

I filtered for this event, then looked at the log:
![](/screenshots/114.png)
![](/screenshots/115.png)
![](/screenshots/116.png)

Using the IP address from the log, I searched the Sysmon index for any additional activity involving the IP. Focusing my search within the 4-8AM timeframe (first batch of activity), I didn't find anything else suspicious other than the successful authentication event. Returning to the Sysmon event ID 3 search, I wasn’t able to find any more events of interest outside of the destination ports. With all leads seemingly exhausted on this front, I checked out the other Sysmon IDs. Referring back to the videos from earlier, 1 is another Sysmon ID I'm told to look into; processes do need to be booted up in order to execute its activities, malware included.

Searching the Sysmon index for this event ID within the 4-8AM timeframe, I peeked into the `CommandLine` field, and noticed a few suspicious commands:

![](/screenshots/117.png)

It's unlikely that anyone at the company would have a reason to be running multiple network information commands such as these, so I suspected a potential threat actor performing network discovery here. However, I needed more info for further validation, so I checked out the `C:\Windows\system32\net1 user Administrator` event:
![](/screenshots/118.png)
![](/screenshots/119.png)
![](/screenshots/120.png)
![](/screenshots/121.png)
![](/screenshots/122.png)

I noticed the `process_current_directory` field is `C:\Users\Administrator\`, meaning this command was executed by the user. Under normal circumstances, a regular user _really_ doesn't need to know information about a root user. Factoring that alongside the RDP activity and other network commands around the same time, it’s become all but certain that the Windows server has been accessed by a threat actor via a successful brute force attack to conduct network discovery tactics. Thus, I ran a search for the `Administrator` user (attacker at this point) to uncover the full list of discovery commands ran:
![](/screenshots/123.png)

The discovery commands the attacker ran include, but are not limited to:
  - whoami
  - quser
  - tasklist

I continued digging deeper for any additional suspicious events. After a while, I determined that there weren’t any more for the first batch of activity, so I moved on to the next batch (Timeframe: 6PM-12AM). Right away, I noticed high activity around the 6PM mark:
![](/screenshots/124.png)

However, when looking at the events generated around that mark, I realized that much of them were just critical Windows services booting up, so I reexpanded the 6PM-12AM timeframe and searched for Sysmon event ID 3 to find any suspicious connections. I saw a few in the destination ports:
![](/screenshots/125.png)

I viewed the event with port 3389 first. Skimming through the information, I see that the attacker has returned:
![](/screenshots/126.png)

Then, I filtered for the port 9999 events. I looked at the first event:
![](/screenshots/127.png)

In the log, I noticed something highly alarming: A PowerShell process was created and used to establish a connection to an unknown IP address (172.20.0.92). I want to see what was done in this PowerShell session; in other words, I want to find all other events that are related to this one. To do so, I need to use the process GUID from the event above in a search of the Sysmon index to correlate events involving this PowerShell session. After detouring for a bit to look at all other events in the Sysmon ID 3 search and not finding anything noteworthy, that's exactly what I did. When checking the list of Sysmon IDs, I find an ID of interest - Sysmon ID 29 (file executable detected), generated when Sysmon detects the presence of a new executable file:
![](/screenshots/128.png)

Taking a look at the log associated with this ID, I discovered a suspicious executable file:
![](/screenshots/129.png)

A key element that immediately renders this executable suspicious to me is that it’s located in the `Public\Downloads` folder, whereas legitimate critical executables would typically be located in a `System` folder. Despite that, it's still possible that this could’ve been a legitimate software download, so to verify further, I grabbed the file’s SHA1 hash & ran it through VirusTotal; no results were yielded, further arousing suspicion about the file.

With the file deemed highly suspicious, I ran its name in a Splunk search and checked the Sysmon event IDs for the existence of ID 1, which indicates the process had been executed. Sure enough, it did exist:
![](/screenshots/130.png)

Filtering for the two events, I checked the oldest one first, where I was able to get more info about the file, most notably its original name:
![](/screenshots/131.png)
![](/screenshots/132.png)

I tried searching up the original filename on Google, but much of the results returned were irrelevant in this context. So I continued digging deeper; I grabbed the process GUID of the suspicious executable from the event above and ran it in a Splunk search where the ‘ProcessGuid’ field is equal to the suspicious executable’s process GUID. By using the executable’s process GUID in a search, this allows me to trace the entire action lifespan of the executable, which turned out to be only a few minutes before it terminated:

With this being a dead end, I went back to the search with Sysmon event ID 1 and the suspicious executable name. There, I checked out the second (top) event, grabbed its process GUID, and ran it in a Splunk search to view its action lifespan. This time, I actually do get some logged activity:


Skimming through all 13 events, this one in particular caught my attention:




Here, the suspicious (now malicious) executable makes a connection to the same unknown IP address mentioned earlier. And, unlike with the first Sysmon ID 1 event, this process doesn’t terminate until 8:54:01.000 PM UTC, about an hour after the process made the connection. Along with everything else that’s been mentioned throughout this section, this likely means that the Apollo executable is a C2 malware and malicious C2 activity has occurred with the suspicious IP address as the culprit. Unfortunately, given the nature of C2 activity, C2 actions are generally only capturable on the network layer via network packet tools such as Wireshark or tcpdump. However, in this case, no such tool was installed on the Windows instance, meaning there’s no way to capture exactly what C2 actions were done. Regardless, I’m still satisfied with the discovery, and finalized my investigative report:


One final note: Sysmon didn’t do a good job at detecting if Windows Defender was disabled. Though considering it did detect the malware running, it’s safe to say that Defender was tampered with in some capacity.

## Home Lab Phase: Report, Alert, & Dashboard Creation
With the investigation finished, I proceeded to build a report, alert, and dashboard for future malware detections. For the alert, I want it to trigger when an Apollo.exe process is executed. To do so, I first created a report named “Apollo.exe Executes” with the following query:

Then, while still in “Edit report” mode, I clicked “Save As” at the top right and selected “Alert”, with the following configuration (I also made sure to change the time range for the report to “All time” in order for the alert to work):


I excluded the SHA256 hash in the query because due to the various setups of newly generated Apollo agents, a different SHA256 hash would be generated for each new agent, thus rendering the field not helpful to use in a Splunk search.
Dashboard Panel #1
For the dashboard, instead of using the authentication dashboard, I created a new one that will house three panels. The first panel will be a table showing all executed malicious processes. This table uses the following fields:
index=”sysmon-events”
EventID=1
ParentImage
Image (specifically, processes running PowerShell, cmd, or rundll32 (which many malware tend to utilize, according to the Day 28 video))
CommandLine
OriginalFileName
ParentCommandLine
User
UtcTime (timestamp)
ProcessGuid
Building the query for the panel:

Then, saving it to a new dashboard with the following configuration:
Dashboard Title: Malware Activity
Description: Left blank
Permissions: Left as Private
Build as “Classic Dashboard”
Panel Title: Malware Executed Events
Visualization Type: Statistics Table
Advanced Panel Settings:
Panel Powered By: Inline Search
Drilldown: No action


Dashboard Panel #2
For the second panel, it will be a table showing all malicious connections. This table will have the following fields:
index=”sysmon-events”
EventID=3
SourceIp
DestinationIp
DestinationPort
Image
UtcTime (timestamp)
Initiated=true
Panel query:

Saving it to the existing Malware Activity dashboard with the panel title “Malware Initiated Network Connection Events”, I then entered dashboard Edit mode. In Edit mode, I clicked on the brush icon on the top right of the panel, then selected “Row” under “Click Selection” to make the data interactable, letting me see more information on a particular row simply by clicking on it:

I also did the same thing for the first panel (hence why the text is blue). When looking at the data in this panel, I noticed a lot of unnecessary excess logs coming from MpDefenderCoreService.exe & svchost.exe, so I figured that while I’m still in Edit mode, I would edit the panel’s query to exclude these two processes:

After applying the changes, this is what things look like now:

Dashboard Panel #3
For the third panel, this will be a table that simply shows which Windows hosts had their Windows Defender disabled. This table has the following fields:
index=”windefender-events”
EventCode=5001 (Windows Defender disabled code)
_time (timestamp)
host
Panel query:

After saving it to the existing Malware Activity dashboard with the panel title “Windows Defender Disabled Events”, I entered dashboard Edit mode. In Edit mode, I added a description for the panel by clicking on the text box directly below the panel title; I also did this for the first panel earlier. For this panel’s description, I inputted the Windows Defender disabled event message (since I couldn’t use it in a table field), then applied the changes:

Converting to Splunk Dashboard Studio
After adding in a time picker & submit button and modifying the panels to use the time picker, the dashboard is now complete. However, due to the high number of fields for the first two panels, they’re so wide that I’d have to use the horizontal scroll bar to see the other fields. I could see a couple of potential problems with doing this when trying to conduct event analysis, one being the tedium from constantly scrolling back & forth to get all event information for a particular row. I tried going into Edit mode to see if I could shrink some of the columns, but Classic dashboard wouldn’t allow me to. I then tried setting “Wrap Results” for these panels to off, but that still didn’t fix the problem.
With no other options that I could think of, I looked up on Splunk’s Dashboard Studio to see if it would enable me to adjust the column sizes. When I found out that it can, I didn’t hesitate to convert the Classic dashboard to Dashboard Studio. Doing so was easy: Click on the three dots button at the top right of the dashboard, then select “Clone to Dashboard Studio”:



Overall, the conversion went exactly as I expected. With the dashboard now converted, I created three new tabs in the new dashboard, one for each panel, then proceeded to move the panels to their respective tabs:



Classic dashboard’s Wrap functionality was the only thing not converted over to Dashboard Studio, but it doesn't matter since Dashboard Studio lets me perform drag resizing for each of the panel columns. After adjusting the column widths, this is my final dashboard for malware activity:



