# Part 13 (The Bonus Challenge): Running a Real Malware and Analyzing its Activity Using Splunk
It has been 12 excruciatingly long parts, but I’ve finally finished MyDFIR’s challenge. This is a journey that's given me a better understanding of core SOC analyst & cybersecurity knowledge, from networking fundamentals to effective threat hunting with the use of a SIEM tool like Splunk. Now, to put everything I've learned from this lab to the test, I’ve thought of one more challenge to try. As I’ve alluded in the beginning of this writeup (and in this part's title), I want to execute an actual piece of malware from the internet in a controlled environment, then analyze its activity using Splunk to try and determine its purpose.

## Preparations
Before starting the challenge, I went through a list of preparations. First, I modified the search query of my malware dashboard, specifically the **Initiated Network Connections** panel:
![](/screenshots/313.png)

Due to Microsoft Defender periodically updating itself, a different version of the software started reflooding my panel with data. The software's version is shown here:

![](/screenshots/314.png)

So, I modified the panel's query to include all versions of the executable. That way, Microsoft Defender connection events will no longer show up, regardless of version.

The next item on the list is a VPN (Virtual Private Network), which will change my host's public IP address while active. This will be needed whenever I connect to the test instance to keep my home network safe from a potential attack. There's no telling what the malware I'm using could do; it could peek at the events under Event Viewer. Considering that's also where the instance logs the IP addresses of machines that attempt to connect, it's possible that I might become a target as a result of negligence. For the VPN, I picked ProtonVPN for this, as it has an actual free tier and is legitimate (the VPN is [open source](https://github.com/protonvpn), so one can view the software’s entire code to verify that no suspicious background shenanigans are happening). I'm also a bit biased here, as I’ve been using it for sometime now, with no issues.

As for how I'm connecting to my test instance, I’m going to do so through my Kali VM from Part 8. The VM will act as another layer of defense should the malware escape the instance via exploiting any network protocol vulnerabilities, adhering to the defense in depth principle. I need NAT selected for bringing internet access to the VM to even establish a connection, which poses a problem since it's still possible for the attacker to discover my host, given how NAT works. In response, I executed least privilege principles on the VM, which included tasks like ensuring shared folders, clipboard sharing, and drag & drop are disabled. In theory, this should minimize the risk of an infection on my host.

As for where I'll run the VPN, I'm running it in Kali, mainly due to the fact that the IP change would mess with my ability to connect to my other instances that are also relevant here (Although in hindsight, I definitely would’ve been able to (and should’ve) run the VPN on my host, provided I change a couple of steps in this part). I followed [this guide](https://protonvpn.com/support/official-linux-vpn-ubuntu?srsltid=AfmBOorByyKOB7STaZCUJXT4OkmLaqIbDaSp6UQ2Tk_7jP7_Of-15W_b) to install ProtonVPN on Kali.

### Playing with Wireshark
The final item on the list is a network packet capturing tool (packet sniffer), which enables monitoring & viewing of network communications for suspicious activity. Particularly, as mentioned in Part 8, this tool can capture the actions executed in a C2 session, making this tool perfect for whatever kind of malware I’m dealing with. Searching the internet for a packet sniffer turns up many different options to choose from, but Wireshark is the most widely recommended, so I went for that.

Having decided upon a packet sniffer, I want to spend some time learning how to use the tool first before I can start applying it for the malware analysis. So, I installed the tool onto my RDP Windows server, then loaded up my Mythic setup and ran the following C2 commands:
- `whoami`
- `ifconfig`
- `download hello-world.txt`
![](/screenshots/315.png)

After running the commands, I checked Wireshark to see how the activity came out. As MyDFIR mentioned in the Day 28 video, any C2 activity such as the one just now would be captured by Wireshark. However, Steven also mentioned that any captured network communications, including C2 traffic, would be encrypted, asides a few exceptions. Sure enough, when following the TCP stream of my packet capture, much of the activity captured came out encrypted, unfortunately:
![](/screenshots/316.png)
![](/screenshots/317.png)
![](/screenshots/318.png)
![](/screenshots/319.png)
![](/screenshots/320.png)

Despite the predicament, I still attempted to make out as much as I could, using the results from the commands as a baseline. For instance, I took note of the following data binary:
![](/screenshots/321.png)

Looking at the timestamp on the server (Mythic instance) stream at the top, the client (Windows server) streamed the data binary to the server around 8\:40\:55 AM GMT time, or UTC time, the same time I ran the `ifconfig` command in Mythic:
![](/screenshots/322.png)
- Note that the timezone for the Mythic instance is 4 hours behind Wireshark.

The POST request, coupled with the unknown IP address (Mythic instance), does provide an IoC (indicator of compromise) for C2 activity, regardless of command context.

The following TCP streams seems to capture a typical connection check in process from Apollo:
![](/screenshots/323.png)

While exploring more of the packet capture, I actually found the following unencrypted packet:
![](/screenshots/324.png)

But when I checked the packet’s assemblies, they were encrypted as well:
![](/screenshots/325.png)

With all these encrypted packets flowing in, a question immediately comes to mind: Is there any way I can decrypt them for inspection? Based on information I found on the internet, performing this task necessitates me acquiring a decryption key for Apollo. To hunt for this key, I tried searching the internet for any publicly leaked keys; the idea was based off [an article](https://isc.sans.edu/diary/27968) I stumbled upon where such a key was used to decrypt Cobalt Strike traffic. Then, based on what I've read from the [Mythic check in documentation](https://docs.mythic-c2.net/customizing/payload-type-development/create_tasking/agent-side-coding/initial-checkin), I opened up the Mythic instance's PowerShell session and ran commands searching the system files for anything containing a key of some sort. I thought that I could find the decryption key within the system files, but unfortunately no results turned up. So next, I went to the configuration settings of my Apollo payload. There, I seemingly found what I was looking for, but when I tried using the key in Wireshark, it didn’t decrypt the files. At this point, I gave up with my search.
- In retrospect, I was kind of a fool here. I should've been looking into the C2 profile for this key, not the config settings of the payload.

Having failed in my goal to decrypt the Wireshark packets, I sought additional internet resources for any information that would let me make sense of them in their encrypted state. I got some good pointers from [this video](https://www.youtube.com/watch?v=ObUgYDn1zZ0). According to it, details that I should watch for include:
- Examining locations and/or IP addresses within the context of a business (is there a server at the location that the business is supposed to communicate with? Is the IP address known to the organization? Is the IP address a known malicious server?)
- Suspicious naming conventions (ex: A server named `cowboy`)
- On older TLS versions, grabbing the JA3 hash and running a search on it
- Looking at unencrypted HTTP traffic

Going back to the decryption keys, I found out there's another key I could've used: A captured session key, as shown in [this video](https://www.youtube.com/watch?v=5qecyZHL-GU). I tried this out for myself, and successfully decrypted the packets of a Wireshark homepage packet capture I had done. Unfortunately, I discovered this technique too late; I didn't enable session capturing when I did the packet capture for Mythic (nor did I have the foresight to enable it for the malware capture later on).

## Setup Process
After spending my time becoming familiar with Wireshark, I felt ready enough to begin this challenge. Since I’ll be dealing with live malware here, I want to create a separate subnet for the infected instance, adhering to the concepts of network segmentation. As such, I adjusted my diagram, mapping out the new addition:
![](/screenshots/326.png)

With everything planned out, I proceeded to create the subnet `infected-subnet`:
![](/screenshots/327.png)
![](/screenshots/328.png)
![](/screenshots/329.png)

Along with its corresponding NACL, `infected-acl`:
![](/screenshots/330.png)
![](/screenshots/331.png)
![](/screenshots/332.png)
![](/screenshots/333.png)

Now that the subnet is set up, I constructed the test instance, providing the following specs:
- Name: Infected-Server
- AMI: Windows, Microsoft Windows Server 2022 Base
  - Architecture: 64-bit (x86)
- Instance type: c7i-flex.large. Since I’m infecting this instance with malware, having high specs would probably ensure consistent performance.
- Key pair: RSA type, .pem format
- Network settings:
  - VPC: MYDFIR-30Day-SOC-Challenge
  - Subnet: infected-subnet
  - Auto-assign public IP: Enabled
  - Firewall (security groups): Create security group, with the following settings:
    ![](/screenshots/334.png)
    ![](/screenshots/335.png)
- Configure storage: 1 volume only:
  - Text box: 30 GiB
  - Dropdown box: Left as whatever option is selected

Afterwards, I connected to the instance through Kali w/ VPN enabled, then proceeded to install the Splunk forwarder, Sysmon, and Wireshark onto the instance. For the forwarder, I decided to have the events forwarded to the same index as my RDP Windows server (`windows-server-events`), as a way of learning how to manage multiple machines in the same index. To differentiate the instances, the hostname that I've given the test instance is **Compromised Windows Server 2022**.

With all the necessary tools set up, I wanted to create a backup for my test instance, as well as for Kali. I managed to make one for the latter, but not for the former; on AWS’s free tier, the best option from what I've read was to create snapshots of my storage volume. However, there’s a catch:
![](/screenshots/336.png)

Given the tools that I’ve installed onto the instance, as well as the system files, those would very likely cause the snapshots to exceed the 1 GB limit, making this option unfeasible for me. With no other viable substitutes, I'm forced to run this malware with no fallback measures in place.

With only one shot to run the malware, I made sure the following are in place before proceeding:
- MYDFIR-Splunk is up & running for log forwarding
- VPN is active in Kali
- An active Wireshark packet capture session in the test instance, with the capture filter `not (tcp port 3389) or (net (<kali_vpn_ip> or 172.20.0.30))` to exclude RDP packets, as well as Splunk & Kali packets, from being captured
  - To determine the public IP in Kali, I ran the command `curl -4/-6 icanhazip.com`
- Windows Defender is turned off in the test instance
- Shared clipboard, drag & drop, and shared folders are disabled for the Kali VM
- Remote connected to the test instance through Kali

## Running the Malware
> [!CAUTION]
> All the sites listed in this section are considered malicious. If planning to analyze any of these sites, do so within a test environment, configured to be completely isolated from other machines within the network (in other words, all potential links between the test environment and the machines are severed) in order to ensure the infection doesn't spread.

After getting everything set up accordingly, it was time to bring in the malware. My first candidate for this was a link from a spam text message I received, but going to that link in the test instance directed me to a car dealer website with nothing interesting. Then, in my search for potential malware samples, I had the idea of checking uBlock Origin’s online malicious URL block list:
![](/screenshots/337.png)

Skimming through the list of candidates, I settled upon the one highlighted:
![](/screenshots/338.png)

Right away, when going to the link, an executable named `view.exe` was downloaded. Running the executable in the test instance, I noticed a few suspicious surface-level activities. First, a couple of applications were automatically downloaded and configured to auto-start upon logging in; one of those is AnyDesk, a remote desktop software. Seeing this app, I disconnected from the instance, thinking my remote session would prevent the attacker from being able to connect via the software. After waiting a few hours to let the malware run its course, I reconnected to the instance, then stopped the packet capture, having felt that I captured all the data I need. Saving the capture as a `.pcap` file, I now needed a way to extract it into Kali without doing any drag & dropping, as to ensure I don't bring the malware to Kali in the process. I was able to pull this off by utilizing Mythic. With `infected-acl` allowing outbound traffic to the Mythic instance, I booted the instance up and downloaded the Apollo binary onto the infected test instance. Then, while navigating to the directory containing the binary, I stumbled upon another surface-level activity involving the `view.exe` executable, this time in the Public directory:
![](/screenshots/339.png)
![](/screenshots/340.png)

Here, some unusual executables and files popped up in the directory. I made a mental note of these files, as I continued on to the directory with the Apollo binary, ran it, then with the Mythic GUI open in Kali (while the VPN is still active), I ran the `download` command, targeting the pcap file. Once the file is successfully downloaded, I navigated to “Search Files” in Mythic and clicked on the pcap file path to extract into Kali.

## Analyzing the Malware’s Activity
Now that I have a copy of the pcap saved into Kali, I set it aside at the moment while I opened up Splunk on my host laptop to begin analyzing the logs generated by the malware. Starting things off, I searched the Sysmon index for events containing event ID 15 (FileCreateStreamHash), which generally occurs via downloads from the web browser, a noteworthy spot for finding suspicious software:
![](/screenshots/341.png)

I grabbed the file path highlighted and ran another search for all events where the event ID is 1 & the process path is the aforementioned file path:
![](/screenshots/342.png)

Through this search, I grabbed the process GUID of the executable to begin correlating events:
![](/screenshots/343.png)

### view.exe Events
Searching the executable’s process GUID returned 16 events involving the executable:
![](/screenshots/344.png)

The events paint the following timeline as to what the executable did (All Splunk times are in UTC):
- 12\:38\:17.951 AM: Executable was executed
- 12\:38\:19 AM: A lot of `.txt` files were created. All of these files were associated with event ID 29, indicating potentially unwanted software that was automatically installed by the executable. For instance, one of the files I noticed was `AnyDesk.txt`, referring to the AnyDesk application that I spotted earlier. The full list are as follows:
![](/screenshots/345.png)
- 12\:38\:19.265 AM: A batch script file, `C:\Users\ADMINI~1\AppData\Local\Temp\2\Start.cmd`, was created. This file was executed almost immediately, on 12\:38\:19.648 AM, by `C:\Windows\SysWOW64\cmd.exe`. Looking up on SysWOW64, I learned that this folder enables 32-bit applications used by `view.exe` to run on 64-bit hardware.
- 12\:38\:19.474 AM: A suspicious dll, `C:\Windows\SysWOW64\urlmon.dll`, was loaded by the executable
- 12\:38\:19.654 AM: Executable was terminated

The GUID trail ran cold at this point, but I did one more thing before moving on. I recalled from the Day 28 video that MyDFIR searched up the process ID of the Apollo executable in an attempt to find more leads. I tried the same strategy with the executable, and actually got an additional lead that I could work with:
![](/screenshots/346.png)
![](/screenshots/347.png)
![](/screenshots/348.png)
![](/screenshots/349.png)
![](/screenshots/350.png)
![](/screenshots/351.png)

The execution of an attribute removal process on `L2cache`, a critical system processor (according to the top sources on the internet), from the `Downloads` directory makes this event suspicious. Furthermore, the system, hidden, and read-only attributes were targeted in the removal. The `parent_process` field unveils another batch script, `Start3.cmd`, for future correlation. In the meantime, I began tracing the activities of `Start.cmd`.

### Start.cmd Events
Searching the process GUID of `C:\Users\ADMINI~1\AppData\Local\Temp\2\Start.cmd` yielded 4 events:
![](/screenshots/352.png)

Excluding the process creation event, the following occurred with this batch script:
- Two registry value set events (event ID 13) targeting `HKLM\System\CurrentControlSet\Services\bam\State\UserSettings\S-1-5-21-490752419-3154468780-1763789047-500\\Device\HarddiskVolume1\Windows\SysWOW64\cmd.exe`. The value was encoded, rendering it impossible to determine what exactly happened. However, given the context on event ID 13 [here](https://www.gravwell.io/blog/whats-in-a-sysmon-event-windows-registry-eventids-12-13-14) and the fact that the target registry involves `C:\Windows\SysWOW64\cmd.exe`, I can only assume that malware persistence is being established here.
- Another batch script, `C:\Users\ADMINI~1\AppData\Local\Temp\2\Start.cmd`, was spawned. This script has a different process GUID compared to the previous `Start.cmd` script:
![](/screenshots/353.png)

### Start.cmd #2 Events
Searching the process ID of `Start.cmd` #1 turned up no additional leads, so I began correlating the other Start.cmd script:
![](/screenshots/354.png)

Event timeline:
- 12\:38\:19.872 AM: A Windows script (wscript) named `C:\Users\Public\testvb1.vbs` was created. This script was executed almost immediately at 12\:38\:19.885 AM.
- 12\:38\:19.889 AM: Another batch script, a variant of `Start.cmd` named `C:\Users\Administrator\AppData\Local\Temp\2\Start2.cmd`, was created. At the same time, the command `C:\Windows\system32\cmd.exe /c type "C:\Users\ADMINI~1\AppData\Local\Temp\2\Start.cmd"` was executed, where `type` displays the contents of `Start.cmd`. Furthermore, a registry set event occurred concurrently, targeting `Start.cmd`.
- 12\:38\:20.196 AM: `Start2.cmd` was executed through PowerShell, running the command `C:\Windows\System32\WindowsPowerShell\v1.0\PowerShell.exe -Command "Start-Process 'C:\Users\Administrator\AppData\Local\Temp\2\Start2.cmd' -windowstyle hidden"`. The `hidden` option keeps the process window for the batch script hidden while active, thwarting attempts to simply close off the script.
- 12\:38\:21.917 AM: A ping operation to a private IP address, 192.168.1.1, was discovered
- 12\:38\:30.816 AM: Another registry set event for malware persistence, targeting `C:\Windows\SysWOW64\cmd.exe`.

No more additional leads came up when searching the process ID, so I proceeded to correlate the scripts that came up from this search, starting off with `testvb1.vbs`.

### testvb1.vbs Events
What I noticed:
- A lot of DLLs were loaded. One of those, `wshom.ocx`, is an old one, dating back to the 2000s.
- On 12\:38\:20.134 AM, the following command was executed: `"C:\Windows\System32\cmd.exe" /c cd /d C:\Users\Public\ &amp; copy /v /b /y C:\Users\Public\testvb1.vbs C:\Users\Public\testvb2.vbs`. Essentially, this command copies the contents of the current Windows script into another one named `testvb2.vbs`.

### Start2.cmd Events
Having hit a dead end with `testvb1.vbs`, `Start2.cmd` was next:
![](/screenshots/355.png)

- A handful of these events involved loaded DLLs. The majority of them had a legitimate signature, with the exception of one:
![](/screenshots/356.png)
![](/screenshots/357.png)
![](/screenshots/358.png)
![](/screenshots/359.png)
- 12\:38\:21.292 AM: The following PowerShell script was created: `C:\Users\Administrator\AppData\Local\Temp\2\__PSScriptPolicyTest_lby1cg5c.kqb.ps1`. But this script was deleted shortly afterwards, without being executed once.
- 12\:38\:21.470 AM: A PipeCreated event (event ID 17) was generated:
![](/screenshots/360.png)
![](/screenshots/361.png)
![](/screenshots/362.png)
[Looking up on Windows pipes](https://learn.microsoft.com/en-us/windows/win32/ipc/pipes), I learned that they enable processes to communicate between each other, either on a host, or multiple hosts across a network. As such, some likely intentions I could think of for this named pipe include:
  - Giving the applications & processes installed by the malware a means of sending sensitive data between/communicating with each other.
  - Spreading the malware infection to other reachable hosts within the VPC.  
  
  I wasn't able to find any conclusive information as to whether they could communicate between processes on hosts across different networks. Not that it matters anyway, as when I searched the pipe's name, no event containing event ID 18 (Pipe Connected) came up, meaning the pipe hadn't been used in any capacity.
- 12\:38\:21.875 AM: A different `Start2.cmd` script was executed
- 12\:38\:21.882 AM: A FileCreate event occurred, targeting `C:\Users\Administrator\AppData\Local\Microsoft\Windows\PowerShell\StartupProfileData-NonInteractive`. An action of `modified` was also specified in the event, indicating additional persistence shenanigans by the malware.
- 12\:38\:46.712 AM: Registry set event targeting Windows Defender’s AllowFastServiceStartup. I'm able to see the value set in this case, which is 0x00000000, or disabled. Under this context, Defender runs with low priority, essentially allowing all components of the malware to activate without Defender getting a chance to take any action.

### Start2.cmd #2 Events
![](/screenshots/363.png)

- 12\:38\:21.947 AM: The `C:\Users\ADMINI~1\AppData\Local\Temp\2\Start.cmd` script was removed by this one. Then almost immediately, another batch script, `C:\Users\Administrator\AppData\Local\Temp\2\NhStart3.cmd`, was created at 12\:38\:21.950 AM.
- 12\:38\:22.219 AM: The registry command `C:\Windows\System32\reg.exe query "HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System" /v "ConsentPromptBehaviorAdmin"` was ran, where `query` displays a list of subkeys and name values under the `HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System` key, with `/v` telling the command to display the value of the name value `ConsentPromptBehaviorAdmin` specifically.
- 12\:38\:22.251 AM: Another registry query command was ran, this time targeting `HKCU\Software\Microsoft\Windows`. Reconnaissance was the first thing that came to mind when seeing these query operations; the attacker is likely trying to change the privileges and configurations of the computer & current user in order to allow the malware to run at its fullest potential.
- 12\:38\:22.263 AM: Another batch script file, `C:\Users\Administrator\AppData\Local\Temp\2\uac.cmd`, was created and ran with the window hidden. Given the filename, I had a hunch to look it up on the internet. According to the top sources, UAC stands for User Account Control, indicating that this file was likely designed for modifying user permissions to enable further elevated privileges for the malware, coinciding with the previous point about reconnaissance.
- 12\:38\:22.614 AM: Ping to 192.168.1.1 detected
- 12\:38\:36.324 AM: `C:\Windows\System32\reg.exe query "HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System" /v "ConsentPromptBehaviorAdmin"` command was repeated
- 12\:38\:36.351 AM: A new `Start.cmd` variant, `Start3.cmd`, was created
- 12\:38\:36.353 AM: The PowerShell command `C:\Windows\System32\WindowsPowerShell\v1.0\PowerShell.exe -Command "&amp; {Get-Content -Path "'C:\Users\Administrator\AppData\Local\Temp\2\NhStart3.cmd'" | Out-File -FilePath "'C:\Users\Administrator\AppData\Local\Temp\2\Start3.cmd'" -Encoding ascii}" -Wait` was ran. This command basically copies the data of `NhStart3.cmd` into `Start3.cmd`, with the data encoded into ASCII via `-Encoding ascii` and `-Wait` keeping the PowerShell process active until the task is completed. Shortly after, `NhStart3.cmd` was deleted, and `Start3.cmd` was started as administrator on 12\:38\:36.820 AM.

### uac.cmd Events
Searching the process GUID of uac.cmd returned 16 events:
![](/screenshots/364.png)

However, I realized that a lot of them were ones that I’ve seen before:
- Suspicious DLL `System.Management.Automation.ni.dll` loaded
- PowerShell script `C:\Users\Administrator\AppData\Local\Temp\2\__PSScriptPolicyTest_d5dstequ.qk0.ps1` created & deleted shortly afterwards
- PowerShell PipeCreated event
- Another `uac.cmd` script executed at 12\:38\:22.569 AM
- `C:\Users\Administrator\AppData\Local\Microsoft\Windows\PowerShell\StartupProfileData-NonInteractive` modified

Because of the similarities, I returned to the process ID strategy, using that of the current `uac.cmd` script to find additional IoCs:
![](/screenshots/365.png)

Sure enough, I uncovered some potential ones:
- 12\:43\:20.923 AM: The following suspicious task was scheduled: `C:\Windows\System32\schtasks.exe /create /sc minute /mo 30 /tn "Svtasks" /tr "\"C:\Users\Administrator\AppData\Local\Temp\2\svtasks.cmd\"" /f`. The task `Svtasks` runs the batch script `svtasks.cmd` every 30 minutes, ignoring warnings if such a task already exists (`/f`).
- 12\:43\:21.170 AM: A DLL, `C:\Windows\SysWOW64\taskschd.dll`, was loaded
- 12\:45\:16.728 AM: Another batch script, `C:\Windows\SysWOW64\show.cmd`, was executed

### uac.cmd #2 Events
For the other `uac.cmd` script, I also used its process ID instead of the process GUID for correlation:
![](/screenshots/366.png)

Notable events:
- 12\:38\:22.620 AM: PowerShell command executed: `"C:\Windows\System32\WindowsPowerShell\v1.0\PowerShell.exe" -Command "Start-Process cmd -ArgumentList '/c "C:\Windows\System32\WindowsPowerShell\v1.0\PowerShell.exe" Set-Itemproperty -Path REGISTRY::HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System -Name ConsentPromptBehaviorAdmin -Value 0' -Verb RunAs -Wait -windowstyle hidden"`. This hidden PowerShell session is started as administrator to disable the `ConsentPromptBehaviorAdmin` registry. This enables the malware to perform elevated operations without needing to seek permissions first, confirming my reconnaissance theory. The process doesn’t stop until its task is completed.
- 12\:46\:02.769 AM: Ping to 192.168.1.1
- Command ran to list all currently running tasks. Either process discovery in action or for verification that the `Svtasks` task exists.

## A Different Approach
As I continued correlating batch scripts, I found myself encountering even more scripts to correlate. By this point, I was growing fatigued and uncertain on how long this chain will go before the interesting stuff finally shows up. So, I changed my approach - I stopped correlating the scripts through process GUIDs, and instead ran a query that shows a table of all the event codes involved after the time the malware was ran:
![](/screenshots/367.png)

From here, I began searching through the event codes that are likely to indicate the presence of malware. I’ve filtered out destination port 3389 (RDP) as there hasn't been any successful attempts from any outside entity when I looked into it.

Going from bottom to top, or least amount of events to highest:

### EventCode 22 (DNS Queries)
Skimming through the list of DNS queries, I see a lot of suspicious ones:
![](/screenshots/368.png)

I spent some time examining each query for additional information. From there, I put together the following map of each query & their respective IP addresses and processes that called them:

| QueryName                      | IP addresses(es) and/or DNS                  | Process(es) involved                                        |
| ------------------------------ | ---------------------------------------      | ----------------------------------------------------------- |
| auth10.aeroadmin.com           | 89.40.115.70                                 | `C:\ProgramData\IntelSvc.exe`                               |
| pro13.emailserver.vn           | :\:ffff:103.110.129.113                      | `C:\Windows\SysWOW64\WindowsPowerShell\v1.0\powershell.exe` |
| ulm.aeroadmin.com              | :\:ffff:104.21.37.227,<br/> :\:ffff:172.67.214.170  | `C:\ProgramData\IntelSvc.exe`                        |
| xemhang.vn                     | :\:ffff:118.70.144.70                        | `C:\Windows\SysWOW64\WindowsPowerShell\v1.0\powershell.exe` |
| auto.c3pool.org                | :\:ffff:154.12.244.206,<br/> :\:ffff:85.239.243.201 | `C:\Users\Administrator\AppData\Local\Temp\2\RtkAudio.exe`  |
| boot.net.anydesk.com           | boot-relays.net.anydesk.com,<br/> :\:ffff:185.229.191.44,<br/> :\:ffff:185.229.190.236,<br/> :\:ffff:92.223.88.7,<br/> :\:ffff:185.229.191.39,<br/> :\:ffff:92.223.88.232,<br/> :\:ffff:92.223.88.41,<br/> :\:ffff:195.181.174.167,<br/> :\:ffff:57.129.19.230,<br/> :\:ffff:57.129.37.75,<br/> :\:ffff:57.129.37.28 | `C:\Users\Administrator\AppData\Local\Temp\2\AnyDesk.exe`,<br/> `C:\ProgramData\AnyDesk\AnyDesk.exe`                   |
| donate.ssl.xmrig.com           | :\:ffff:45.32.185.122,<br/> :\:ffff:178.128.242.134 | `C:\Users\Administrator\AppData\Local\Temp\2\RtkAudio.exe`  |
| relay-dc7501e7.net.anydesk.com | :\:ffff:92.38.177.14                         | `C:\Users\Administrator\AppData\Local\Temp\2\AnyDesk.exe`,<br/> `C:\ProgramData\AnyDesk\AnyDesk.exe` |
| EC2AMAZ-DH1SHEO                | fe80:\:c3a3\:6a8\:aaff:38e,<br/> :\:ffff:172.20.0.248,<br/> 172.20.0.248 (machine IP) | `C:\Windows\SysWOW64\wbem\WmiPrvSE.exe`,<br/> `C:\ProgramData\AnyDesk\AnyDesk.exe`,<br/> `C:\ProgramData\IntelSvc.exe` |
| auth11.aeroadmin.com<br/> auth14.aeroadmin.com | 127.0.0.1                    | `C:\ProgramData\IntelSvc.exe`                               |
| auth17.aeroadmin.org           | N/A                                          | `C:\ProgramData\IntelSvc.exe`                               |

Throughout this process, I’ve uncovered a couple of suspicious processes that I haven’t seen before, most notably IntelSvc.exe and RtkAudio.exe. I’ll get to those momentarily.
EventCode 6 (Driver Loaded)
I uncovered a suspicious image with the path C:\Users\Administrator\AppData\Local\Temp\2\WinRing0x64.sys:

While the image itself is legitimate, the format of the file path is the exact same as the paths for the Start.cmd series of batch scripts, marking this as a potential indicator of compromise.
EventCode 1 (Process executed)
A lot of processes to deal with:


For the sake of time, I’m only going to focus on the ones that I’ve deemed suspicious:
icals.exe: Executed at 12:38:48.358 AM by the Start3.cmd script previously uncovered. The command ran was C:\Windows\System32\icacls.exe "C:\Windows\System32\smartscreen.exe" /grant:r Administrator:F, which grants the Administrator user full access to Windows Defender, replacing any previous privileges. A likely intent for doing this is to permanently remove Defender with ease, or at the very least disable it.
takeown.exe: Executed at 12:38:48.343 AM by Start3.cmd, with the command C:\Windows\System32\takeown.exe /s EC2AMAZ-DH1SHEO /u Administrator /f "C:\Windows\System32\smartscreen.exe". This is related to the icals.exe event, as this makes the Administrator user the owner of the Windows Defender internal files.
wscript.exe: Executed at 12:38:19.885 AM by Start.cmd. The script executed was C:\Users\Public\testvb1.vbs, which I’ve already covered.
RtkAudio.exe: Executed at 12:41:19.365 AM by Start3.cmd. I ran its SHA256 hash through VirusTotal, which is where I discovered that this is a masquerading malicious process:

According to the analyses left by the top security vendors, one thing was constantly mentioned about this xmrig.exe executable: Cryptomining. This means that the view.exe executable has cryptomining intent as one of its malicious tasks. I decided to try searching up the executable name on Google, and the top results returned the homepage and the GitHub repository:

I then performed log correlation for the executable, and discovered that it ran the command "C:\Users\Administrator\AppData\Local\Temp\2\RtkAudio.exe" --config="C:\Users\Administrator\AppData\Local\Temp\2\Tweaker\desktops.ini". According to the XMRig documentation, this command loads the JSON-formatted configuration file desktops.ini.
net.exe: Executed 3 times by Start3.cmd within the timespan of 12:38:46.370 AM - 12:45:15.966 AM. The following commands were executed:
C:\Windows\System32\NET.exe stop windefend, which stopped Windows Defender, as I predicted
C:\Windows\System32\net.exe stop "Service Network"
C:\Windows\System32\net.exe start "Service Network" a few seconds after the previous command was ran. The two commands stopped, then restarted a Windows service named Service Network. Searching this exact service name on Google didn’t turn up any conclusive information, yet I found some for something similar, Network Service, which ranges from a Windows process to a user account. The start action here only furthers the ambiguity on the purposes of this service.
screen.exe: Executed by Start3.cmd at 12:41:33.879 AM. Original name of this executable was NirCmd.exe, and the command ran was "C:\Users\Administrator\AppData\Local\Temp\2\screen.exe" win hide ititle "RtkAudio", which hides the process window for the cryptomining software uncovered earlier. A different one located in the C:\Users\Public directory was also executed at 12:42:53.837 AM by taskmgr.cmd, with the command win hide ititle "kmgtas", hiding the window of a process named kmgtas.
AnyDesk.exe:
12:40:40.834 AM: Executed by Start3.cmd; installed via the command line. A couple of  options were specified during the installation process; the first option, --silent, ensured a covert installation, while the second option, --start-with-win, enabled the application to start automatically.
12:40:41.333 AM: Another command was run, this time with the --local-service option specified. This tells AnyDesk to use the client app to start up with limited functionality if it can’t connect to the service that the software uses. This was likely done for persistence purposes.
fontdrvhots.exe: Originally named IntelSvc, I ran its SHA256 hash through VirusTotal. It turns out that this process is flagged as malicious:


Using the process GUID, I searched for all events involving this process. Only 6 events were returned, and I also noticed that all events occurred an hour apart from each other. This strongly indicates that an automated script is involved, which only became more plausible when looking at the parent process command: C:\Windows\system32\svchost.exe -k netsvcs -p -s Schedule. So, I searched the process name itself in an attempt to find this script, and did:


Additionally, I noticed the netsh.exe command directly above the highlighted scheduled task, which adds a firewall rule named Intel3 that allows inbound IntelSvc traffic. This also caught my interest, so I took a look at its log:


The logs tell me that both processes were executed by Start3.cmd. Given everything else I’ve uncovered so far, it’s evident that this batch script is responsible for setting up the core components of this malware, so I grabbed its process GUID and process ID for future correlation:

taskkill.exe: Ran commands that forcibly ends Task Manager and Windows Defender Smartscreen as well as the child processes started by them (/t option; only in the case of Task Manager). However, it also forcibly ends the cryptominer process, which I find strange:

sc.exe: Executed by Start3.cmd. Commands ran:

All of these commands disable critical Windows services, enabling full reign of the malware, as aforementioned.
IntelSvc.exe: A quick peek at the CommandLine field reveals that the process was installed as a service:

The parent process for this process was a PowerShell session, to which the parent process for that PowerShell was Start3.cmd.
schtasks.exe: 
Commands ran:

The /Change commands disables all Windows tasks that help protect the computer automatically, relating to the purposes of sc.exe.
netsh.exe: TCP ports 50001, 6568, & 80 were opened on the machine, and firewall rules were created to allow traffic from tv_x86.exe and TeamViewer.exe, alongside other suspicious executables already mentioned. The full list of commands:


powershell.exe: Interesting bits:
-ExecutionPolicy: Enabled a couple of suspicious .ps1 scripts to run without any caveats.
-inputformat -outputformat -NonInteractive -Command Add-MpPreference -ExclusionPath: Excluded various directories housing the malware components from being scanned for malware.
-Command DownloadFile(): Used to download a file named recoverdv.txt from the same domain where the malware was first delivered.
-Command Get-Content: Made encoded copies of some .txt files and .cmd batch scripts (including Start3.cmd), making them harder to examine manually.
CreateShortcut(): Creates a link file named TeamViewer_Service in the Windows Startup folder, allowing tv_x86.exe to automatically run at startup. Typing this, I realized that tv_x86.exe is actually TeamViewer.
Set-ExecutionPolicy: Allowed all malicious scripts to run.




cmd.exe:
The command C:\Windows\system32\cmd.exe /c reg query "HKLM\SYSTEM\ControlSet001\Services" /s /k "webthreatdefusersvc" /f 2&gt;nul | find /i "webthreatdefusersvc" finds all currently active Windows Defender services, likely for the malware to disable to ensure no thwarting.
Two echo commands caught my interest: /S /D /c" echo Thuyhue1234" and /S /D /c" echo "svchost#24" 2&gt;NUL". The second seems to target a specific svchost process, while the first looks to be some sort of username/password (part of the text is also present in the domain I visited to download the malware).
Full list of commands:


tasklist.exe: Suspicious process Sophos.exe uncovered
findstr.exe: Some of the commands grab information from .txt files created by the malware, likely to be piped into other operations as part of the malware setup process. The full list of commands:

PING.exe: The full list of commands:

I was clueless as to why all these commands targeted this one IP address, so I went and searched it up on Google. This is where I discovered another potential target of this malware:




According to these results, as well as the Apple thread in the last picture, the private IP address 192.168.1.1 is used to access the admin portal of a router. This means that these pings are indicative of a potential network takeover operation by the malware. Thankfully, with how I set up my NACL, this didn’t work.
Having seen a lot of these malicious processes initiated by Start3.cmd, I used the script’s process GUID and began correlating events. Running it in a search returned 500+ events, which are all the malware setup events. So, I took a look at the parent process that spawned this script. The parent process was Start2.cmd, then that parent process was Start.cmd, and finally that parent process was the view.exe executable itself. Throughout the analysis process, I also drew a diagram of the hierarchy for better visualization:

EventCode 3 (Network Connections)
I want to look at all established outbound connections from the infected machine to an outsider IP, so I ran the following search to list all the destination IPs involved:


Then, after modifying the query to remove everything to the right of the pipe character (|) & the pipe itself, I checked the Image field. There were 7 images that made a network connection of some sort. Most I already know were malicious, but two of them were seemingly legitimate:
C:\ProgramData\Microsoft\Windows Defender\Platform\4.18.25080.5-0\MpDefenderCoreService.exe.
C:\Windows\System32\svchost.exe.
I took a look at the events involving the first executable. Checking the dest field, I find the following IPs involved:

Running the IPs through AbuseIPDB, VirusTotal, and GreyNoise reveals that they’re all legitimate IPs used by Microsoft, hence MpDefenderCoreService.exe is legitimate. So, I modified the query again to exclude the process from the search, as well as focusing only on established connections:

Now, these are the images involved:

I narrowed down the query further to return only the events involving svchost.exe, and took a look at the IPs, both source and destination:


I ran these IPs (excluding 172.20.0.248 since that’s the machine itself) through AbuseIPDB and GreyNoise, as well as Google. This is what I found:
224.0.0.251 is the IP used for mDNS
0:0:0:0:0:0:0:1 is the IPv6 localhost address
fe80:0:0:0:c3a3:6a8:aaff:38e is a private IP address, according to AbuseIPDB
ff02:0:0:0:0:0:0:fb didn’t return anything in AbuseIPDB, but according to GreyNoise, this is a private IP address:

While none of these IPs were labeled as suspicious, I wasn’t willing to accept this conclusion yet, after what I uncovered with ping.exe. I continued digging deeper; I checked the event codes for each IP, but nothing out of the ordinary came up, so I checked the destination ports:

According to this documentation, (UDP) port 5353 is used for mDNS, which svchost.exe typically connects to, and is less likely to be used by malware since it can be easily detected through this route. Because of that, I figured it’s safe to exclude events containing this port, which leaves port 5985. After some searching on Google, I discovered that this port is used for WinRM (Microsoft Windows Remote Management), with 5985 used for HTTP and 5986 for HTTPS. Looking up WinRM, I came across this article, which has a section describing the advantages of WinRM. In this section, something caught my attention:

Realizing that there might be a script executed by this process that I haven’t caught, I immediately went on the hunt for this event. I filtered for the port to determine the timestamp I would need to base my search around. Despite returning two events, both share the exact timestamp:

Then I ran another search on the Sysmon index for all Compromised Windows Server 2022 events that occurred 1 minute before & after 12:39:34.444 AM UTC. This brought up a lot of events, so I did some filtering to bring this number down into something manageable. First, I tried searching for events where the Image field is System, but got a set akin to the previous screenshot:

A little more filtering later, I came across the echo “svchost#24” command again, which I already found in the EventCode 1 analysis:


Here, I decided to try running a query for events with both “svchost” and “24”, hoping that I could find a svchost.exe event with the 24 pointing to a process ID number. 27 events came back, but none had a ProcessId value of 24. All other filters I’ve tried hit a dead end; I can only assume at the moment that this port 5985 event is for AnyDesk and/or TeamViewer.
As for the other IPs in this screenshot:

Those were associated with DNS queries uncovered in the EventCode 22 analysis. For example, 57.129.37.28 is associated with the DNS query boot.net.anydesk.com triggered by the AnyDesk.exe process. The IPv4 address even matches the IP address appended to ::ffff: exactly:

EventCode 17 & 18 (PipeEvent; Pipe Created & Pipe Connected)
Searching for event code 18 returned 12 events:

All events involved the image C:\Windows\System32\svchost.exe, the same one I tried examining for suspicious activity in the EventCode 3 analysis. in the screenshot above, I noticed that the PipeName in both events starts with TSVCPIPE. I searched this name up on Google to see if I can find anything noteworthy, and I came across this article, particularly this section:

Additional context on virtual channels from a previous section of the article:

In the diagram, the remote machine represents my infected Windows server, while the client machine represents the attacker’s machine. Based on the information provided, the existence of a connected pipe beginning with the name TSVCPIPE indicates the prevalence of some remote activity. However, the timestamps of the events closely match the timeframe I connected to the instance to stop the Wireshark packet capture. As such, these events appear to be legitimate and shouldn’t be concerning.
With event code 18 turning up nothing suspicious, I searched the event code 17 logs. 64 events in total were returned, with the following images involved:

Coincidentally, C:\Windows\System32\svchost.exe here has the same number of events as event code 18, and the exact same timestamps as well:

Which means the other processes didn’t have their pipes utilized in some capacity. The Wireshark.exe event came from me, as the timeframe matches the time I started another capturing session:

As for AnyDesk.exe, I read through the AnyDesk documentation, and realized that the software uses modules that can initiate and/or receive connections, depending on the type:

In this case, then theoretically speaking, if a successful connection was made, an event ID 18 should’ve been generated for the AnyDesk pipe. But since no such event exists, it’s likely the attacker hadn’t connected to the instance through the software yet. This brings up a point of contention: The successful event code 3 events for AnyDesk.exe. These events are likely referring to the aforementioned service that the software connects to.
EventCode 10 (ProcessAccess)
AnyDesk.exe happened to be the process responsible for a majority of these events:


Filtering for Explorer.EXE events, I find that both target AnyDesk.exe, with the same call stack. This call stack shows some of the DLLs involved:

Next, I filtered for the AnyDesk.exe events. These are the images that this executable targeted:

Others
In addition to everything above, I’ve also looked into event code 7, as well as the reg.exe process. However, when re-examining my analyses for both, it turned out that everything I uncovered I’ve already uncovered elsewhere, such as the suspicious images for event code 7. As such, I don’t have any intention of going in-depth for both. These were all the suspicious images in event code 7 (the ones with no signature):

And the processes that loaded them:

For reg.exe, there were so many commands to go through in a timely manner:



Using Wireshark to find more Information
Now that I’ve gone through every interesting event code to the best of my ability, I delved into the packet capture to see if I can find any additional information that wasn’t captured by Splunk, mainly any potential C2 activity. Unfortunately, without a decryption key in hand, making sense of the encrypted packets proved to be extremely difficult with my current skill level (as of writing this). Though, I did find some unencrypted information:



I decided to try searching the packet capture for the IP addresses and DNS queries I uncovered from looking at the event codes, in an attempt to uncover additional information that the Splunk logs missed. For the most part, the results were similar to what I’ve found from looking at the event codes, but I found a couple interesting points:
When searching the IP 154.12.244.206, I may have stumbled across an unencrypted password for XMRig:

I searched up the aeroadmin.com domain, and found out that Aeroadmin is also a remote desktop application, designed for streamlining the connection process to a remote computer.

