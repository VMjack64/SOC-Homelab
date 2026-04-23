# Part 6: Getting Logs Ingested Into Splunk (Day 10)
With everything all set up, it’s finally time for me to get the `windows-server-events`, `windows-sysmon-events`, and `windefender-events` indexes filling up with logs. To accomplish this, I need an `inputs.conf` file in the Windows UF's internal files, which contains the stanzas telling the UF to forward the logs to their corresponding index. I plan on using two different methods to add the file: The direct method for `windows-server-events`, and the deployment server method for `windows-sysmon-events` and `windefender-events`.

## Stanza Creation
### Index: "windows-server-events" (Direct route)
To forward the base Windows event logs to the `windows-server-events` index, I need to manually add an `inputs.conf` file under the `C:\Program Files\SplunkUniversalForwarder\etc\system\local` directory (`local` directory for short). Opening this file in Notepad, I added the following stanza text:
![](/screenshots/31.png)

As the file didn't exist under `local`, I copied the one from `C:\Program Files\SplunkUniversalForwarder\etc\system\default` (`default` directory for short). I don't want to modify the one under `default`, as this file gets overwritten with a new version whenever the UF is updated. As a result, the UF now has multiple `inputs.conf` files in place.

To handle the presence of multiple `inputs.conf` files, Splunk has a precedence system in place, where it processes the file in the following directories, from top to bottom:
![](/screenshots/32.png)

Where `disabled = 0` in the second screenshot above is an example of an attribute and `App` also includes deployment server apps.

> [!NOTE]
> Depending on the context of the Splunk configuration file & setup, a different precedence system can be used. More on this can be found [here](https://help.splunk.com/en/splunk-enterprise/administer/admin-manual/10.0/administer-splunk-enterprise-with-configuration-files/configuration-file-precedence#id_1913f2f7_28ce_4520_8a77_cff1caa5d5ca__Configuration_file_precedence).

### Index: “windows-sysmon-events” & “windefender-events” (Deployment server route)
For the other two indexes, I need to use my deployment server to push more `inputs.conf` files to the UF. I wanted to do it this way instead of doing the manual route as a way to see Splunk’s precedence system in action.

With the deployment server instance on, I connected to it via SSH, then created the following directories (apps in this context) by running the following `mkdir` commands:
  - `mkdir $SPLUNK_HOME/etc/deployment-apps/sysmon-event-logs`
  - `mkdir $SPLUNK_HOME/etc/deployment-apps/windefender-logs`

Where $SPLUNK_HOME = `/opt/splunk`.

After creating the apps, I made the server class next:
1. In the deployment server's web GUI, I navigated to “Settings” > “Agent management”.
2. On the “Server Classes” tab, I clicked on “New Server Class”.
3. After entering in a name for the server class, I clicked “Save”.
4. On the server class’s screen, I clicked on the dropdown arrow next to the “Edit forwarders” button, then selected “Edit applications”.
5. I assigned the two apps I just created to the server class by checking all the apps, then clicking the “>” button on the screen.
6. After saving the changes, I navigated to the “Forwarders” tab, then clicked “Edit forwarders”.
7. Under “Include (required)", I specified the private IP address of the Windows server (`172.20.0.214/32`), then clicked “Save”.
8. On the "Agent management" screen, I navigated to the “Applications” tab, then clicked on the text for the first app.
9. I enabled the options `Enable application` & `Restart agent`, then clicked on “Applications” on the top left to navigate back to the “Agent management” screen (any changes are automatically saved).
10. I repeated steps 8 and 9 for the second app.

Once the server class has been established, I went back to the PowerShell SSH terminal, changed to the `sysmon-event-logs` directory, then created an `inputs.conf` file in the app’s `default` directory with the following stanza:
![](/screenshots/33.png)

Then, I did the `windefender-logs` app:

![](/screenshots/34.png)

## Ingesting Time! What Can Go Wrong?
Before firing up the ingesting process, I need to enable receiving for “MYDFIR-Splunk” real quick. While indexing capabilities are active by default in newly created Splunk Enterprise instances (as mentioned in Part 2), log receiving capabilities aren’t. To enable receiving for “MYDFIR-Splunk” through the Splunk web GUI, I performed the following steps:
On the Splunk web GUI, I clicked on “Settings” > “Forwarding and receiving”.
Then, I clicked the “Configure receiving” link.
On the “Receive data” screen, I made sure there aren’t any existing receiver ports that are enabled, since a duplicate receiver port can’t be created. After confirming there aren’t, I clicked “New Receiving Port”.
I need to specify a port number next to “Listen on this port” and click “Save”. Conventionally, the receiving port used by indexers is 9997 by default, so I just went with that.
With the stanzas set up & receiving enabled on the main Splunk instance, it’s now time to get things running for real. After making sure the Windows Server instance is on, I ran the command ‘./splunk reload deploy-server’ in the PowerShell SSH terminal for the deployment server Ubuntu instance. In doing so, the following chain of events gets set off:
The deployment server is reloaded, and all connected agents (in this case, so far, the Windows UF) poll the server for any changes to get.
The Windows UF is then told it’s part of a server class with the ‘sysmon-event-logs’ & ‘windefender-logs’ apps, and is expected to have them in its ‘C:\Program Files\SplunkUniversalForwarder\etc\apps’ directory. However, the UF checks this directory, but doesn’t find either of these apps. So the UF grabs a copy of the apps & everything in them from the deployment server and stores them in the UF’s ‘apps’ directory.
Since each app is configured to restart the agent, the UF is automatically restarted for each app it grabs.
With the UF restarted, it begins looking through the ‘inputs.conf’ files in its system directories, following the configuration file precedence system. The files, now updated to contain the stanzas from the “Stanza Creation” section of this writeup, tell the UF to forward logs to the main Splunk instance.
To see if the logs are being ingested into Splunk successfully, I went to the Splunk web GUI for “MYDFIR-Splunk”, navigated to the “Indexes” screen, then scrolled down to my three indexes to see if each one has a log count that’s not 0, which indicates that log ingestion for that index is working as expected. Right away, I do see that’s the case for the “windows-server-events” index:

I was unsure on how I could check out the logs for myself though, so I turned to my Splunk lesson notes, and from the “Getting Data Into Splunk” lesson, I can do so by running a search query for the index, so I did, and the logs seem to look fine:

So, that’s that. Nothing went wrong throughout this process, and I can go start analyzing some malicious activity, right?
Home Lab Phase: …Something Did Go Wrong, After All
Well, no. While the “windows-server-events” index did work as expected on the first try, the same couldn’t be said for the other two indexes. Just to make sure it wasn’t because I missed something in the documentation stating that Splunk couldn’t handle config files located in app directories grabbed from a deployment server, I copied the stanzas for the "windows-sysmon-events" & “windefender-events” indexes to the ‘inputs.conf’ file in ‘system/local’, then manually restarted the UF and checked “MYDFIR-Splunk”’s indexes to see if logs were being ingested. They were not, which indicates that there might be a problem in regards to the stanza settings.
After rolling back the changes from the previous paragraph, I spent some time troubleshooting the issue, or rather, running googling searches to find the solution to my problem. Along the way, MyDFIR’s Basic Home Lab series came to mind. One of the things I remembered doing in that lab was setting up Splunk & Sysmon on a Windows VM (virtual machine) and using them to analyze a metasploit attack simulated via Kali Linux. One step in particular entailed modifying the ‘inputs.conf’ file to add a stanza telling Splunk to ingest Sysmon logs, which is where I immediately realized that the answer to my problem could be there, so I opened up Part 3 of the series and skipped to the part where Splunk’s ‘inputs.conf’ file is modified, which is where I see Steven’s settings for the Sysmon stanza, as well as the Windows Defender stanza:

Immediately, the first difference I noticed was that both stanzas have a ‘source’ attribute, with the full path to the logs specified. Another difference I also noticed was that the Sysmon stanza has a ‘renderXml’ attribute, which tells Splunk to convert each Sysmon log into XML format during the ingestion process. From what I read online, the reason for converting Sysmon logs to XML is because of compression; each base Sysmon log takes up more storage than expected, so when there’s a lot of logs being ingested (which can easily happen, depending on how Sysmon is configured), the index can hit the storage cap pretty quick, thus preventing more logs from being ingested. By converting the logs to XML, the amount of storage is reduced for each log, ensuring that the index doesn’t run out of storage quickly.
After navigating to my apps in the deployment server SSH PowerShell terminal to add these attributes to my Sysmon & Windows Defender stanzas, I reloaded the deployment server to echo these changes to the UF, then checked “MYDFIR-Splunk”’s indexes on the web GUI to see if the additions fixed the problem. Right away, I see that it did work for Windows Defender logs: 

From the looks of it, I think the reason why making the changes fixed the ingestion for Windows Defender logs is because Splunk couldn’t seem to determine the location of the Defender logs just from the stanza declaration alone, unlike with the Windows Application, Security, and System logs. I think that the space between “Windows Defender” may have something to do with it.
Home Lab Phase: One Problem Fixed, Another One Arises
While the changes fixed the ingestion of Windows Defender logs, Sysmon logs were still not being ingested, so I had to spend even more time troubleshooting / googling. After countless internet resources & forums, I stumbled upon this post on Reddit that describes my exact situation. A few people responded with potential solutions; I tried the most upvoted solution (as of writing this) first, but it didn’t help, since each step was either not applicable to my setup, or just didn’t work. However, this solution caught my attention:

I checked the debug logs for the UF in the ‘splunkd’ file under ‘C:\Program Files\SplunkUniversalForwarder\var\log\splunk’ to see if I was getting this type of error, and sure enough, permission issues were indeed the cause of my problem:


Though I’ve finally figured out the reason why Sysmon logs aren’t being ingested, fixing it was another matter. The solution provided by the Reddit user would essentially entail running the UF as the root user, which is bad practice, as previously established, so I did even more searching to figure out how I could solve this problem without needing to do any rooting shenanigans. Which is when I came across this community post on the Splunk forums, most notably this post:

The user explains the mechanics behind my Sysmon ingestion issue in a concise manner: Essentially, the UF operates as a “normal user” account by default, but Sysmon logs are considered “high value event logs”, which are logs that can only be accessed by privileged accounts. So, I can access these logs manually since I’m logged into the instance on an Administrator account (which is a privileged account on Windows), but as things currently are, the UF can’t access the Sysmon logs. Alongside the explanation, the user also provided a potential solution: Modify the “channelAccess” settings to grant the UF access to the Sysmon logs without giving the UF account privileged status. Unfortunately, I quickly found out that this solution required knowledge with SDDLs (Security Descriptor Definition Languages), which proved to be too complicated for me to grasp at the moment. So, I continued scrolling down the post, and found this solution just seconds later:

This one seemed to be much simpler to implement compared to the last solution, so I opened up Group Policy (by searching for it on the Windows instance’s taskbar search), but I couldn’t find the Event Log Readers group in there. I wasn’t gonna give up on this solution, though, so I googled where I could find the “Event Log Reader” group, and Google’s AI Overview pointed me towards Windows Computer Management:

I decided to follow the overview on a whim and opened up Computer Management. Sure enough, I actually found the group that I was looking for:

Now to add the UF to the group. I double-clicked on the group, clicked “Add…”, then under “Enter the object names to select”, I entered in the UF’s system name. I found this info by doing the following:
Navigate to “Services” by searching for it on the taskbar search, then selecting what appears under best match.
Scroll to the “SplunkForwarder” service. To navigate to it even quicker, click on a service, then press the “S” key.
After finding the “SplunkForwarder” service, right click on it, then click “Properties”.
Sidenote: Steps 1-3 can also be imitated to manually restart or stop the UF.
Click the “Log On” tab
Copy what appears in the “This account:” field. This is what will be entered under “Enter the object names to select”. In my case: 

After entering in the UF’s system name, I clicked “Check Names” to ensure that the name exists (which should be a given, considering I just searched for it) & the correct name for the UF is given. Then, I clicked OK to exit the “Select Users” window, then OK to exit the “Event Log Reader Properties” window. With all the changes applied, I manually restarted the UF, then checked “MYDFIR-Splunk”’s indexes to see if Sysmon logs are being ingested… and they are, at long last:


