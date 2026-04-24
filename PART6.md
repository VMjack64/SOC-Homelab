Previous Part                                    | Return to Introduction                  | Next Part
------------------------------------------------ | --------------------------------------- | ---------
[Part 5: Prepare for Log Ingestion](/PART5.md)   | [Introduction](/README.md#introduction) | [Part 7: The First Step Towards Becoming a Proefficient Splunk User](/PART7.md)

[Part 6: Getting Logs Ingested Into Splunk](#part-6-getting-logs-ingested-into-splunk-day-10)
- [Stanza Creation](#stanza-creation)
  - [Index: "windows-server-events"](#index-windows-server-events-direct-route)
  - [Index: “windows-sysmon-events” & “windefender-events”](#index-windows-sysmon-events--windefender-events-deployment-server-route)
- [Ingesting Time! What Can Go Wrong?](#ingesting-time-what-can-go-wrong)
- […Something Did Go Wrong, After All](#something-did-go-wrong-after-all)
- [One Index Fixed, The Other Still Malfunctioning](#one-index-fixed-the-other-still-malfunctioning)

# Part 6: Getting Logs Ingested Into Splunk (Day 10)
With everything all set up, it’s finally time for me to get the `windows-server-events`, `windows-sysmon-events`, and `windefender-events` indexes filling up with logs. To accomplish this, I need an `inputs.conf` file in the Windows UF's internal directories, which contains the stanzas telling the UF to forward the logs to their corresponding index. I plan on using two different methods to add the file: The direct method for `windows-server-events`, and the deployment server method for `windows-sysmon-events` and `windefender-events`.

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
Before firing up the ingestion process, I need to enable receiving for MYDFIR-Splunk real quick. While indexing capabilities are active by default (as mentioned back in [Part 2](/PART2.md#key-pointers-on-deployment-servers), log receiving capabilities aren’t. To do so, on MYDFIR-Splunk's web GUI:
1. I naivgated to “Settings” > “Forwarding and receiving”.
2. Then, clicked the “Configure receiving” link.
3. On the “Receive data” screen, if there aren’t any existing receiver ports enabled (since a duplicate receiver port can’t be created), select “New Receiving Port”.
4. Next to “Listen on this port”, I specified the port number 9997 and clicked “Save”.

With that out of the way, the stage is all set for things to get running for real. After verifying that the Windows server is turned on, I opened up the PowerShell SSH session for the deployment server, changed directories to `$SPLUNK_HOME/bin`, then ran the command `./splunk reload deploy-server`. In doing so, the following chain of events is set off:
1. The deployment server is reloaded, and all connected agents poll the server for any changes to get.
2. The Windows UF is then told it’s part of a server class with the `sysmon-event-logs` & `windefender-logs` apps, and is expected to have them in its `C:\Program Files\SplunkUniversalForwarder\etc\apps` directory. However, when the UF checks this directory, it doesn’t find either of these apps. So the UF grabs a copy of the apps & everything in them from the deployment server and stored in the UF’s `apps` directory.
3. Since both apps are configured to restart the agent, the UF is automatically restarted for each app it grabs.
4. With the UF restarted, it begins looking through the `inputs.conf` files in its internal directories, following the global context precedence system. The files, now updated to contain the stanzas from the [`Stanza Creation`](#stanza-creation) section, tell the UF to forward the target logs to their appropriate indexes in MYDFIR-Splunk.

To see if the logs are being ingested successfully, I opened up MYDFIR-Splunk's web GUI, navigated to the “Indexes” screen, then scrolled down until I could find the three aforementioned indexes. I checked to make sure that each index has a log count that’s not 0, indicating that index is working as expected. Right away, I do see that’s the case for the “windows-server-events” index, showing promise that everything worked out fine:
![](/screenshots/35.png)

I further verified the integrity of the logs by running a search query for the index, and the logs look nothing out of the ordinary, as I was expecting:
![](/screenshots/36.png)

## …Something Did Go Wrong, After All
Unfortunately, that promise didn't last long. While the `windows-server-events` index worked fine, the same couldn’t be said for the other two indexes. Just to make sure I didn't misconstrue the precedence system documentation, specficially on what counts as an app directory, I copied the stanzas for the `windows-sysmon-events` & `windefender-events` indexes to the `inputs.conf` file in the UF's `local` directory, then restarted the UF directly and re-checked MYDFIR-Splunk’s indexes to see if the problem was fixed. It wasn't, which meant to me that the problem might lie within the stanza settings.

After rolling back the changes from the previous paragraph, I spent some time troubleshooting the issue, or rather, Googling to find the solution to my problem. During this time, MyDFIR’s Basic Home Lab series came back into mind. I recalled an activity in [Part 3](https://www.youtube.com/watch?v=-8X7Ay4YCoA&pp=ygUVbXlkZmlyIGJhc2ljIGhvbWUgbGFi) of that lab where I set up Splunk & Sysmon on a Windows VM (virtual machine) and using them to analyze a Metasploit attack simulated via Kali Linux. One step in particular entailed modifying the `inputs.conf` file to add a stanza telling Splunk to ingest Sysmon logs, which is when I realized that the correct stanza for my situation might be there, so I opened up Part 3 of that series and skipped to where I can see the stanzas for Sysmon and Windows Defender in the `inputs.conf` file:

![](/screenshots/37.png)

Right away, the first difference I noticed was that both stanzas have a `source` attribute, with the full path to the logs specified. The other difference was that the Sysmon stanza has a `renderXml` attribute, which tells Splunk to convert each Sysmon log into XML format during the ingestion process. From what I read online, there is a valid reason for doing this: Compression. Each Sysmon log takes up more storage than expected in their base state, so when there’s a lot of logs being ingested (which can happen depending on how Sysmon is configured), the index can hit the storage cap pretty quick, thus preventing more logs from being ingested. By converting the logs to XML, the amount of storage is reduced for each log, ensuring that the index doesn’t run out of space quickly.

Going back to the PowerShell SSH terminal for my deployment server, I went to add these attributes to my Sysmon & Windows Defender stanzas, then reloaded the deployment server to echo the changes to the UF. Checking MYDFIR-Splunk’s indexes, I saw that the change fixed the problem for Windows Defender logs: 
![](/screenshots/38.png)

I'm purely making assumptions here, but I think the reason why the changes fixed the ingestion for Windows Defender logs is because Splunk couldn’t seem to determine the location solely from the stanza declaration, unlike with the Windows Application, Security, and System logs. The space between `Windows Defender` may have something to do with it.

## One Index Fixed, The Other Still Malfunctioning
The changes may have fixed the Windows Defender index, but the Sysmon index was still not working, so I spent even more time troubleshooting. Many internet resources & forums later, I stumbled upon [this Reddit post](https://www.reddit.com/r/Splunk/comments/1jxjkvo/splunk_not_taking_in_sysmon_source/) describing my exact predicament. A few people responded with potential solutions. I tried the most upvoted solution (as of writing this), but it didn't help, since each step was either not applicable to my setup, or just didn’t work. However, this other solution caught my attention:
![](/screenshots/39.png)

A dive into the debug logs for the UF, located in the `splunkd` file under `C:\Program Files\SplunkUniversalForwarder\var\log\splunk`, verified that permission issues were indeed the cause of my problem:
![](/screenshots/40.png)
![](/screenshots/41.png)

Although I’ve finally figured out the cause of the Sysmon log ingestion problem, fixing it was another matter. The solution provided by the user would essentially entail running the UF as root, which is bad practice, so I did more searching to figure out how I could fix this problem without the need of any rooting shenanigans. That's when I came across [this community post](https://community.splunk.com/t5/Getting-Data-In/Sysmon-events-not-getting-indexed/m-p/655609:) on the Splunk forums, more specifically this post:
![](/screenshots/42.png)

The user concisely explains why my Sysmon ingestion issue is happening as such: Basically, the UF operates as a “normal user” account by default, but Sysmon logs are considered “high value event logs”, which are logs that can only be accessed by privileged accounts. This means I can access these logs manually since I’m logged into the Windows server as Administrator, a privileged account on Windows, while the UF can’t. Alongside the explanation, the user also provided a viable solution. However, in trying to implement it, I found the underlying knowledge complicated for me to grasp at the moment. So, I continued scrolling down the community post, hoping to find an easier solution, and found this:
![](/screenshots/43.png)

Using the taskbar search in the Windows server instance, I searched for & opened up **Group Policy**, but I didn’t find the **Event Log Readers** group in there. I wasn’t giving up on this solution, though, so I Googled where I could find the Event Log Reader group, and was pointed towards Windows Computer Management (thanks Google AI Overview). Opening up Computer Management, I actually found the group I'm looking for:
![](/screenshots/44.png)

Double-clicking on the group, I clicked “Add…”, then under “Enter the object names to select”, I inputted the UF’s system name. I found this info by doing the following:
1. I navigated to “Services” by searching for it in taskbar search, then selecting what appeared under best match.
2. I scrolled to the “SplunkForwarder” service. To get to it even quicker, click on a service, then press the “S” key.
3. After finding the “SplunkForwarder” service, I right clicked on it, then click “Properties”.
> [!NOTE]
> Steps 1-3 can also be imitated to manually restart or stop the UF.

4. Click the “Log On” tab.
5. I copied what appeared in the "This account:" text box, which will be pasted under “Enter the object names to select”. In my case:
![](/screenshots/45.png)

After entering in the UF’s system name, I clicked “Check Names” to ensure the correct name for the UF is given. Then, I clicked OK out of all windows and applied the changes. Afterwards, I manually restarted the UF, then checked MYDFIR-Splunk’s indexes hoping that Sysmon logs are being ingested… and they are, at long last:
![](/screenshots/46.png)

Previous Part                                    | Return to Introduction                  | Next Part
------------------------------------------------ | --------------------------------------- | ---------
[Part 5: Prepare for Log Ingestion](/PART5.md)   | [Introduction](/README.md#introduction) | [Part 7: The First Step Towards Becoming a Proefficient Splunk User](/PART7.md)
