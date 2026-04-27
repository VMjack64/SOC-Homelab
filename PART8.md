# Part 8: Taking Things Seriously with Mythic (Days 18, 19, 20, 21, 22, 28)
Having spent a good amount of time searching logs & creating dashboard panels with Splunk, I definitely have gotten to the point where I have a grasp on the software fundamentals. With that, I took things to the next step and began analyzing some malware. Specifically, I’m going to run an Apollo executable on the Windows instance and try to analyze its activity with Splunk.

## Attack Preparations
Before running the exectuable, I need to do some preparation work. The first task on the list is to construct a brute force attack diagram outlining how the attack is going to play out:
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
> To avoid any legal ramifications, only perform these attacks on machines that YOU own.

With all the necessary preparations done, I began the process of simulating the attack in the diagram from the previous section. After connecting to the Mythic instance via SSH & updating its repositories, I was able to install Mythic onto the instance by following the Day 20 video without any caveats. Then, I followed along with the Day 21 video to execute the attack, but with a couple of alterations:
1. I’ve opted to stick with the default Windows server password provided upon instance creation time instead of changing it, since I thought AWS will automatically roll back the changes.
2. I’ll be using the `hello-world.txt` file I created back in the Log Analysis Warmup section, instead of creating a `passwords.txt` file.

In trying to follow along with the video, I ran into a problem with Crowbar. When running the tool, I get the following error:

`xfreerdp: /usr/bin/xfreerdp path doesn't exists on the system`

A straightforward issue at first glance, I went and tried to install xfreerdp onto Kali, but was having a difficult time finding the package. I eventually found out that the package is obsolete and replaced by xfreerdp3 in the Kali repositories. Yet, Crowbar still needed the old xfreerdp executable for RDP to work (at the time of writing this). To fix this conundrum, I installed the xfreerdp3 package, then created a symlink to the executable named `xfreerdp` and moved the symlink file to the `/usr/bin/` directory.

Afterwards, I reran the Crowbar command, and this time it successfully executed. Unfortunately, I immediately ran into another problem: Crowbar was not able to find the password in the file. I double checked a few things:
  - The file to confirm it’s saved with the Windows instance password already inputted.
  - The terminal to make sure I’m running the command in the same directory location as the file.
  - The public IP address of the Windows instance on AWS to make sure it’s the correct one.
  - The username on AWS (in this case, ‘Administrator’) to make sure it’s correct.

Even after verifying all of these, Crowbar still couldn’t find the password after running the command again. When I tried debugging the command, I noticed that Crowbar seemed to be stuck in a loop. So at that point, I just abandoned Crowbar and checked out other similar tools. Eventually, I tried out Hydra, which was mentioned in the Day 11 video, and ran the following command in the same directory as the `.txt` file specified:

`hydra -l Administrator -P mydfir-wordlist.txt rdp://[<public-ip-of-windows-server>/32]:3389`

When running the command the first time around, Hydra didn’t find the password, just like with Crowbar. However, I immediately ran it again (as I assumed the failure was due to improper asynchronization), and this time it actually worked:
![](/screenshots/104.png)

With the “brute force” attack successful, I connected to the instance from Kali via xfreerdp(3) to complete the first phase.

> [!NOTE]
> When specifying the password in the `xfreerdp` command, it’s recommended to enclose it in parentheses, since the password might contain special characters that can mess with the command formatting.

After connecting to the Windows server, I executed the next couple of phases of the attack. First, I performed host & network discovery by referring to [this site](https://car.mitre.org/analytics/CAR-2016-03-001/) for the types of discovery commands that attackers typically use. Then, after disabling Windows Defender, I navigated to the Mythic web GUI in Kali to build the payload that will be implanted into the server.

After generating the payload, I ran the `wget` command in the Mythic SSH terminal to grab the payload, making sure to replace the IP address in the link with the private IP address of the Mythic instance. I repeated this replacement scheme when running the `Invoke-WebRequest` command in Windows, since I have the attacker and target instances in the same VPC. After downloading & running the payload, I checked its status and saw a `SYN_SENT` reading, indicating the connection wasn’t established. Right away, I saw the problem:
![](/screenshots/105.png)

During payload building, I specified the callback host as the localhost IP address (127.0.0.1). The localhost is a loopback IP address to the machine itself. In this context, the payload is trying to connect to a Mythic C2 server on the Windows instance, not the Mythic one. Therefore, I had to repeat the steps of generating the payload, this time using the private IP address of the Mythic instance as the callback host.  downloaded the new payload onto the Windows instance; along the way I found out that the Mythic instance’s Python server should be ran in the same directory as the payload, otherwise I get the following error when attempting to download:

After downloading & running the new payload, I checked its status and saw an ESTABLISHED status, indicating the connection was successful. I didn’t even have to specify ‘ufw allow 80’ on the Mythic PowerShell CLI.
With the Mythic C2 established, I ran a couple of discovery commands, then tried to download the “hello-world.txt” file using the ‘download’ command in the C2 web interface. Despite specifying the absolute path to the file, Mythic was somehow having a hard time finding the file. Eventually, after a few more failed attempts, I decided to try a detoured strategy using ‘ls’, and through it, I was finally able to get the file:



Home Lab Phase: Investigating Mythic C2 Activity in Splunk
With the attack outlined in the diagram successfully carried out, I went and checked out the telemetry generated in Splunk so I can begin investigating it. For this analysis, I wanted it to be as close to a real-life scenario as possible; this assumes absolutely no knowledge on things like what malicious activity was performed, the malicious process(es) involved, or connection method to the Windows instance. To facilitate this, I assume that I’m in the role of a L1 SOC analyst for a company. I boot up my computer & see a message from my boss / coworker:
“There has been reported suspicious activity on the Windows Server around September 10. Please look into it.”
At first, I had no clue where to start, so I watched a few videos on malware log analysis, including the Day 22 & Day 28 videos of the series and this video for some ideas. The last video, in particular, gave me a good starting tip for finding potential malware activity: Look for network connection events (Sysmon event ID 3). Usually, in the case of malware, a network connection would be involved in some capacity, whether it’s initiating a download of the malware from the internet or the malware executing attacks remotely, such as the case in a C2. would make some sort of network connection in order to be able to execute its malicious attacks. Additionally, malware is typically downloaded from the internet as a result of social engineering tactics; a network connection to a server IP address would be involved in the download request initiation.
Running a search for Sysmon event ID 3 on the date specified above, the first I noticed right away is unusually high activity around the 4AM mark on the timeline:

After selecting the 4AM mark to narrow the events down to that timeframe, I checked out the fields present on the left sidebar next, starting from smallest to largest number of unique values. When I checked the dest_port field, I right away noticed a large number of events involving port 3389, an indicator that a brute force attack via RDP occured:

Switching over to the base Windows event logs index, I check the Windows Event IDs and see something that’s cause for concern - event ID 4624 exists:

Narrowing down the event logs to those with Windows Event ID 4624, I looked at the Logon_Type field, and see that a logon type ID of 3 (network, which RDP connections can use) exists, which is very alarming since it indicates that the brute force attack was successful:

Clicking on the ID, I took a look at the event log:



Using this information, I searched the Sysmon index to try and find any additional activity involving the IP address above. Filtering down the events to the same timeframe (4-8AM), nothing else popped up other than the successful connection activity. Returning to the Sysmon event ID 3 search, I also wasn’t able to find anything else noteworthy other than the destination ports. With no more leads on that front, I began checking out the other Sysmon event IDs. Going back to the videos mentioned earlier, I find that another Sysmon ID worth searching for is 1, since processes do need to be started up in order to execute its activities, malware included.
After running a search on the Sysmon index for Sysmon event ID 1 & narrowing the timeframe down to 4-8AM, I checked the CommandLine field, and immediately noticed a few suspicious commands:

Suspecting network discovery tactics in play here, I checked out the ‘C:\Windows\system32\net1 user Administrator’ event log for any information that I could use to uncover the full list of discovery commands used:





I noticed that the process_current_directory field is ‘C:\Users\Administrator\’, and the User field is ‘EC2AMAZ-HP60UKG’, meaning that the command was executed by the Administrator user, which is one of the users (and the root/only user) on the Windows instance. Under normal circumstances, a user shouldn’t be using net commands to uncover information about the root user, let alone a normal one, meaning that it’s very likely the instance has been accessed by an unknown outside entity, in which the culprit most certainly has to be the one who successfully brute forced the instance. With this info, I ran a search for the Administrator user (attacker at this point):

Through this search, I was able to uncover all of the discovery commands the attacker ran, including, but not limited to:
whoami
quser
tasklist
I continued searching deeper to see if I could find any more suspicious activity. After a while, I determined that there weren’t any more events of interest for this timeframe, so I moved on to the next timeframe with activity, 6PM-12AM. Right away, I noticed high activity around the 6PM mark:

Checking the logs in the 6PM mark, I realized that a lot of them were just critical Windows services booting up, so I reexpanded the 6PM-12AM timeframe and searched for Sysmon event ID 3 to find any suspicious connections. Which is when I checked the destination ports and saw a few suspicious ones:

Checking the event log with port 3389, I see that the attacker has returned:

Checking the event logs with port 9999, I checked out the first event:

Looking at the log, I notice something very alarming: A PowerShell process was created and used to establish a connection to an unknown IP address (172.20.0.92). With no other events of interest that I could find from this search, I referred to an idea proposed in the Day 28 video of the series regarding malware event correlation: Grab the process GUID from the log above, use it in a search on the Sysmon index, and then checked the Sysmon event IDs, which is where I find an event ID of interest:

Here, I see a Sysmon event ID of 29 (file executable detected). Looking it up, this event is generated when Sysmon detects the presence of a new executable file, whether it’s from downloading, copying, or renaming the file. Coupled with the suspicious connection, I had to take a look at the log associated with this event ID. This is where I discover a suspicious executable file:


A pointer I got from the Day 28 video: A key element that immediately renders this executable suspicious is that it’s located in the “Public Downloads” folder, whereas legitimate critical executables typically would be located in a “System” folder. Even then, this logic might be wrong; it could’ve been a legitimate software download, so to verify further, I grabbed the file’s SHA1 hash & ran it through VirusTotal; no results were yielded, further arousing suspicion about the file.
With the file deemed highly suspicious, I ran it in a Splunk search and checked the Sysmon event IDs to see if ID 1 exists, indicating the process had been executed. Sure enough, it did:

Clicking on ID 1, I checked the first (bottom) event associated with it. From there, I was able to get more info about the file, most notably its original name:


Searching up the original filename on Google however, returns results that are irrelevant in this situation. So I continued digging deeper; I grabbed the process GUID of the suspicious executable from the event above and ran it in a Splunk search where the ‘ProcessGuid’ field is equal to the suspicious executable’s process GUID. By using the executable’s process GUID in a search, this allows me to trace the entire action lifespan of the executable, which turned out to be only a few minutes before it terminated:

With this being a dead end, I went back to the search with Sysmon event ID 1 and the suspicious executable name. There, I checked out the second (top) event, grabbed its process GUID, and ran it in a Splunk search to view its action lifespan. This time, I actually do get some logged activity:


Skimming through all 13 events, this one in particular caught my attention:




Here, the suspicious (now malicious) executable makes a connection to the same unknown IP address mentioned earlier. And, unlike with the first Sysmon ID 1 event, this process doesn’t terminate until 8:54:01.000 PM UTC, about an hour after the process made the connection. Along with everything else that’s been mentioned throughout this section, this likely means that the Apollo executable is a C2 malware and malicious C2 activity has occurred with the suspicious IP address as the culprit. Unfortunately, given the nature of C2 activity, C2 actions are generally only capturable on the network layer via network packet tools such as Wireshark or tcpdump. However, in this case, no such tool was installed on the Windows instance, meaning there’s no way to capture exactly what C2 actions were done. Regardless, I’m still satisfied with the discovery, and finalized my investigative report:


One final note: Sysmon didn’t do a good job at detecting if Windows Defender was disabled. Though considering it did detect the malware running, it’s safe to say that Defender was tampered with in some capacity.
Home Lab Phase: Report, Alert, & Dashboard Creation
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



