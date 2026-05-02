# Part 10: A Late Linux Addition (Days 12, 13, 14, 26)
Having accomplished integrating osTicket with Splunk, I can finally start doing some in-depth analysis of all the brute force telemetry generated… almost. A reminder of my learning goals for this lab made me decide at the last moment to create a Linux server with the following specifications:
  - Name: MYDFIR-Linux-SSH
  - AMI: Ubuntu, Ubuntu Server 24.04 LTS
  - Architecture: 64-bit (x86)
  - Instance type: t3.micro. Like the Windows RDP server, this one doesn’t need much processing power for its tasks.
  - Key pair: RSA type, .pem format
  - Network settings:
    - VPC: MYDFIR-30Day-SOC-Challenge
    - Subnet: unsecure-subnet
    - Auto-assign public IP: Enabled
    - Firewall (security groups): Select existing security group; choose `unsecure-firewall` in the dropdown under **Common security groups**
  - Configure storage: 1 volume only:
    - Text box: 30 GiB
    - Dropdown box: Left as whatever option is selected

I constructed the instance and left it running exposed to the internet while doing the osTicket part, generating a day’s worth of SSH authentication logs in advance. Afterwards, I swapped the Linux SSH server’s firewall for `Windows-Server-Firewall` to secure the instance, then modified the `30-Day-MYDFIR-SOC-Challenge` firewall to add inbound rules allowing traffic from the Linux server, similar to how I did things for the Windows server:
![](/screenshots/174.png)

## Installing Linux UF
Starting off the list of tasks for the Linux server, I worked on installing the UF onto it. The installation process is similar to the one from the [Splunk Server Setup](/PART3.md#splunk-server-setup) section, but with a few steps changed:
  - 1: Navigate to the "Choose your Download" screen for the universal forwarder instead of Splunk Enterprise. Get the 64-bit `.deb` wget link under the "Linux" tab.
  - 3: The Linux forwarder is installed under `/opt/splunkforwarder` by default.
  - 5: I ran into a problem here, where the forwarder wouldn’t automatically run under `systemd` without root privileges. The Splunk documentation provides a solution for this, which requires Polkit rules:
    ![](/screenshots/175.png)
    Conveniently, the Polkit library came pre-installed on the instance:
    ![](/screenshots/176.png)
    So, I followed this section of the documentation closely, and the problem was remedied.
  - 6: Allow port 8089 instead of 8000.
  - 8: Ignore; Splunk forwarders have no web GUI.

## Adding Linux UF to the Deployment Server and Connecting to the Splunk Indexer
With the UF installed & running, I added it to the deployment server. In the PowerShell session for the Linux instance, I created a `deploymentclient.conf` file under `/opt/splunkforwarder/etc/system/local`, and added the following stanza to the file:
```
[deployment-client]

[target-broker:deploymentServer]
# Specify the deployment server; for example, "10.1.2.4:8089".
targetUri= <URI:port>
```
Where `<URI:port>` = \<private ip address of deployment server instance\>:8089.

Then, I restarted the forwarder by running the command `./splunk restart` in the `/opt/splunkforwarder/bin` directory. If successful, the Linux server should appear in the deployment server’s agent management screen.

Once the Linux UF is added to the deployment server, I connected the UF to the Splunk indexer next. I created (or edited) the `outputs.conf` file under `/opt/splunkforwarder/etc/system/local`, adding the following stanzas to the file:
```
[tcpout]
defaultGroup=index1

[tcpout:index1]
server=<private ip address of MYDFIR-Splunk (instance housing the splunk indexer component)>
```
These stanzas tell the UF to forward all of the Linux server's event logs to the `main` index over at MYDFIR-Splunk. However, I want to send the authentication logs over to its own index by adding an `inputs.conf` file with the following stanza:
```
[monitor://<absolute_path_to_auth_logs>]
disabled = 0
index = <name_of_custom_index>
```
Where `<absolute_path_to_auth_logs>` is `/var/log/auth.log` (or `/var/log/secure`, depending on the Linux operating system) and `<name_of_custom_index>` = `linux-ssh-events`.

I've opted to use my deployment server to add this file. I was able to follow [the same process for setting up the `windows-sysmon-events` index from the Stanza Creation](/PART6.md#index-windows-sysmon-events--windefender-events-deployment-server-route) section here, creating an app directory called `linuxssh-event-logs` and configuring a new server class to associate the app & Linux UF with. After inputting the stanza, I’ve made sure to establish the `linux-ssh-events` index over at MYDFIR-Splunk first before reloading the deployment server to echo the changes to the Linux UF and begin ingesting Linux server events to Splunk.

## A New Problem
With the Linux server’s event logs successfully being ingested, I viewed the authentication events… only to spot a problem almost instantly:
![](/screenshots/177.png)
![](/screenshots/178.png)
![](/screenshots/179.png)

The problem here doesn’t lie within the logs, but rather in the **Interesting Fields** column: There’s no username or IP address field. For some reason, Splunk has an issue extracting these fields from the Linux events naturally. To resolve this, the first solution that came to mind was using Splunk’s field extractor utility by clicking on “Extract New Fields”. After going through the events and looking for patterns, I used the RegEx (regular expression) operation to perform the extractions for source IP address:
![](/screenshots/180.png)

Connection events (invalid or accepted):
![](/screenshots/181.png)
(Tried using `\b` initially, but Splunk had a problem parsing that, so I used `\s` instead)

And usernames:
![](/screenshots/182.png)

The first two produced 100% accurate extractions, but not the third. This could potentially pose a threat to an organization, where an event may be missed during searching, and if this event actually turned out to be a real threat & they successfully got into the internal network, it’d cause havoc for the organization. Hence, I wanted _every_ event parsed correctly.

In an effort to get the usernames parsing properly, I turned to another solution: Installing the **Splunk Add-on for Unix and Linux**. As a prerequisite, I edited the `inputs.conf` file under the `linuxssh-event-logs` app directory to add `sourcetype = linux_secure` (alternate name for Linux logs) under the `[monitor://<absolute_path_to_auth_logs>]` stanza. After saving, then reloading the deployment server to echo the changes, I went to MYDFIR-Splunk’s web GUI to install the add-on from the store.

Although I've installed the add-on, it doesn’t parse the authentication logs from the Linux server's `/var/log` directory by default. I’ll need to enable this parsing by navigating to “Apps” > “Manage Apps”, locating the add-on, clicking “Set Up”, then enabling `/var/log`. For best practice, I also want to disable app visibility by clicking “Edit properties” for the add-on, then setting "Visible" to `No`, making the add-on invisible to other Splunk apps.

After saving the changes, I re-checked the logs, and noticed that not only has the sourcetype changed, but I also get some new fields, particularly the ones I'm looking for:
![](/screenshots/183.png)
![](/screenshots/184.png)
![](/screenshots/185.png)

### Event Re-Indexing
Of course, similar to what occurred when changing the hostname of the Windows server, the change to the sourcetype field doesn't apply to events that have been ingested already. But in this case, I actually do need to update the field for these logs, otherwise the add-on won’t be able to parse them. Looking up on how to accomplish this, my objective here is to re-index the authentication logs. I did a couple of preparations beforehand: First, I again modified the `inputs.conf` in `linuxssh-event-logs` to specify a hostname for the Linux server to appear in Splunk; I've added the `[default]` stanza to the file, with the `host` attribute specified beneath it. Next, I made a backup of the `linux-ssh-events` index in case something went wrong during the re-indexing process by simply copying the index’s database folder, `/opt/splunk/var/lib/splunk/linux-ssh-events`, to the home directory.

After preparations, I began the re-indexing process. The first solution I tried involves deleting the fishbucket index from the UF, after removing all ingested data from the `linux-ssh-events` index. Once the fishbucket index was removed, I restarted the UF. When I viewed the `linux-ssh-events` index in Splunk, I was caught off guard to see only a small amount of ingested events than I was expecting. I looked into the Linux server's `auth.log` file if the old batch of logs was still there, but they were actually erased from the file. I honestly have no clue how this deletion could’ve happened; I wasn't able to find any information about it on the internet. Possible theories I thought of include a side effect from executing the command `./splunk clean eventdata -index linux-ssh-events` in MYDFIR-Splunk's PowerShell session when removing the ingested data, or each event in the `auth.log` file being automatically removed upon ingestion.

With the backup still in hand, I pressed on and searched the internet for another solution. It didn't take me long to find another promising one (thanks again Google AI Overview):
![](/screenshots/186.png)

While I want the old logs re-indexed into `linux-ssh-events`, I don’t want to delete any newly ingested ones from the index, so I skipped step 2 here.
Other than that, I followed the rest of this process, first copying the backup into a new index called `messed-up-linux-events`, then on step 1, ran the command `./splunk search "index=messed-up-linux-events sourcetype=auth" -output csv -maxout 0 > ~/source-type-auth.csv` in MYDFIR-Splunk's PowerShell terminal, then the command `./splunk add oneshot ~/source-type-auth.csv -sourcetype linux_secure -index linux-ssh-events -host "Ubuntu Server 24.04"` for step 3.
  - By default, the time range for the `.csv` file is set to “All Time”, per the Splunk documentation.

Afterwards, I checked the `linux-ssh-events` index if the process was a success, and it pretty much is:
![](/screenshots/187.png)
![](/screenshots/188.png)
![](/screenshots/189.png)
![](/screenshots/190.png)

> [!NOTE]
> Theoretically, I could’ve skipped the entire re-indexing process by specifying a new sourcetype under “Settings” > “Source Types” named `auth` with the `linux_secure` typing. By the time I realized this however, I was already too neck deep into the re-indexing task.

## Creating Reports, Alerts, and Dashboards
Now that all the Linux server logs are parsed & providing additional fields, I can finally begin building reports, an alert, and a dashboard for SSH authentication events. After spending a good amount of time checking all the available fields and figuring out which ones map to what, these are the fields/queries that I used to fulfill the following:
  - Username: `user`
  - Source IP address: `src`
  - Successful authentication: `vendor_action=Accepted`
  - Failed authentication: `tag=start action!=started`. The most optimal query for this would’ve been `action=blocked`, but attempting to search this yields no results:
    ![](/screenshots/191.png)
    Referring back to this diagram from Part 2:  
    ![](/screenshots/3.png)  
    The problem very likely lies within the search-time process (the part between the indexing and searching phases), not within the parsing/indexing phase, otherwise all of the events wouldn't have an `action` field to begin with. Additionally, some searches, such as the one I ran when checking the results of the re-indexing process after ingesting the `.csv` file, did return events with the `action=blocked` label.
      - Looking back, having read through the add-on installation instructions closely, I think that the cause of this problem lies within where I have and haven't installed the add-on onto. I've installed the add-on onto MYDFIR-Splunk (which houses the search head, indexer, and parsing components), but not onto the Linux UF. Considering my Splunk setup is a single-instance deployment with universal forwarders, [the documentation](https://docs.splunk.com/Documentation/AddOns/released/Overview/Singleserverinstall) states that I'm supposed to deploy the add-on to the UF as well. I just got a little confused whilst navigating through the installation instructions, admittedly.

### Report Query and Alert (Failed Authentications Only)
![](/screenshots/192.png)
![](/screenshots/193.png)
![](/screenshots/194.png)
![](/screenshots/195.png)
![](/screenshots/196.png)
Body (HTTP Alert Action):
![](/screenshots/197.png)
I tried many ways to get newline formatting to show up in osTicket, but none of them stuck. Even trying `|||` as a separator for each field doesn't get rendered by osTicket, so I ended up going with a bunch of spaces after each field.

### Map (Failed Authentications)
![](/screenshots/198.png)
Saved to the **Authentication Activity** dashboard with the panel title **SSH Failed Authentication Activity Map**

### Map (Successful Authentications)
![](/screenshots/199.png)
Saved to the **Authentication Activity** dashboard with the panel title **SSH Successful Authentication Activity Map**

### Table (Failed Authentication Activity)
![](/screenshots/200.png)
![](/screenshots/201.png)
Regarding the empty user column under United States, India, and Russia, those refer to these types of events:
![](/screenshots/202.png)
![](/screenshots/203.png)
As well as some events where the user ` ` was specified.

### Table (Successful Authentication Activity)
![](/screenshots/204.png)

Both tables are saved to the **Authentication Activity** dashboard with the panel titles **SSH Failed Authentication Activity Table** and **SSH Successful Authentication Activity Table**, respectively

### Adding Event Logs to Dashboard
During the construction of the table for failed SSH authentication activity, I came across some unusual events, like the ones with the user ` `. These events led to some unorthodox results, as briefly shown in the screenshots there. I worried about the heightened potential of error in human judgement, specifically false negatives, as a consequence of these results, so I wanted to remedy this by adding the failed & successful SSH authentication event logs to the dashboard. Due to Splunk reports utilizing their own time pickers, I'm not able to insert reports into the **Authentication Activity** dashboard as panels. Thus, I resorted to adding "Events" panels, which I could paste the queries into.

For the first event panel, I entered the following information:
![](/screenshots/205.png)

For the second event panel, I entered the following information:
![](/screenshots/206.png)

The current state of my **Authentication Activity** dashboard after adding the panels:
![](/screenshots/207.png)
![](/screenshots/208.png)
![](/screenshots/209.png)
![](/screenshots/210.png)
![](/screenshots/211.png)
![](/screenshots/212.png)

## It’s Investigating Time!
With an updated dashboard on hand, it’s finally time to start the real show: Actually doing some log investigating! I’ll be focusing on the SSH authentication activity in this part, but I will deal with the RDP activity in the next part. Before getting things rolling, I did a couple of preliminaries:
I exposed the Linux instance to the internet throughout September 22 to generate additional telemetry to analyze.
After removing the test tickets, I generated two tickets in osTicket manually by running two curl commands in MYDFIR-Splunk’s CLI:




These tickets are where I’ll report the results of my analyses on each type of brute force authentication activity: One for SSH, the other for RDP.
I ran some failed SSH connection commands to spawn some telemetry that has never been generated before by the attackers:




With those out of the way, I began the investigation process by admitting that I had no idea where to start off, so I turned to the Day 26 video for advice. From there, I got some good starting questions, including:
Any successful attempts? If so, what did the attacker do?
Is the IP address known to perform brute force attacks?
What usernames did the suspicious IP address target?
Additionally, I managed to conjure up a question of my own, befitting of the circumstances from using AWS:
What are the outcomes of the various types of failed login attempts?
Question 1: Any successful attempts (not from my IP address)? If so, what did the attacker do?
Starting off the investigative process with the most obvious question (in hindsight), I took a quick glance at the SSH successful authentications on my dashboard. I do see some successful activity, but all of them came from my laptop’s IP address:


And just to verify that the query in my dashboard isn’t wrong, I ran a query searching for the word “Accepted” in the logs. I ended up getting exactly the same events:


So no, there hasn’t been a successful attempt from any outside entity.
Question 2: What activity did I perform in my first successful authentication session?
There may not have been any successful activity from an outsider, but I figured I’d answer this question while I’m still checking the successful log activity, just for the sake of practice. Grabbing the timestamp for the bottom-most event from my successful authentication report, which is the connection time for my first session, I ran a search for events after that time, with the reverse command piped in to sort the events from oldest-newest, so that I can start the analysis chain from the accepted event:


Right away, I can see some logs that are absolutely unnecessary to my investigation, so I started crafting up some filters that can get rid of the excess noise. First off, I dealt with the “session opened” & “session closed” events, which comprised the bulk of the noise. This returns a significantly lower log count:

Even then, there are still too many logs to be able to analyze in a feasible manner. However, this search also accounts for my activities from the other login sessions, meaning I had to have disconnected from the instance at some point in the first login session, which would’ve been captured by the Splunk UF. To find this event, I modified the query to include my IP address. Thanks to the query, I was able to find the event almost immediately, which is the third log down the list:

With the disconnect event’s timestamp on hand, I modified the query again, this time adjusting the time range to end at the disconnect time for the first login session, as well as removing my IP address from the query:

With this query, only events from the first login session are returned. And fortunately, the amount returned is feasible enough to work with. Combing through the logs, this is the activity I uncovered from my first login session:
Ran the commands apt-get update & apt-get upgrade (repository updating)
Utilized the dpkg command to initiate the installation of a splunkforwarder Debian package. No event of the package being downloaded is recorded; this would be recorded via network packet captures instead of Splunk.
The nano command was ran to create a deploymentclient.conf file in the splunkforwarder‘s local folder, then was subsequently removed
chown was ran recursively on the splunkforwarder directory to change its ownership to ubuntu, which is the main username on the instance
ufw allow was ran twice, one to allow traffic through ports 8089 and again for ports 9997 (the default ports used by splunkd and Splunk receivers, respectively)
Question 3: Is the IP address known to perform brute force attacks? And what usernames did the suspicious IP address target?
There’s a lot of IP addresses for me to work with when viewing all the failed SSH authentications on my dashboard. I could’ve chosen to answer this question for all of them, but for the sake of time, I’ve opted to just use the one with the most events. First, getting the easy stuff right out the gate: A quick glance at the dashboard is all I need to determine the users targeted:

In the Day 26 video, Steven provides a couple of resources that can be used to determine if an IP address is known to conduct brute force attacks: AbuseIPDB and GreyNoise. Grabbing the top IP address and running it through AbuseIPDB, I get my answer almost right away - yes:



For additional intel, I ran the same IP through GreyNoise. The site does in fact reveal it to be an IP known to perform brute forcing:


Question 4: What were the outcomes of the various types of failed login attempts? (In other words, explain each type of failed authentication log)
Because of the fact that I authenticate into my Linux instances in AWS by providing a keyfile instead of a password, this is a good question that came to mind, as I would have a different set of failed authentication logs compared to what I would have if VULTR was my cloud provider instead. To figure out all the types of failed authentication events that have been logged by Splunk, I spent a good amount of time playing around with the list of interesting fields extensively. These are all the types of failed event logs I could find that were generated by my Linux instance, along with a brief explanation as to how they were generated, based on what I could find on the internet:
banner exchange: Invalid format - scanner (see: https://security.stackexchange.com/questions/281647/what-are-these-sshd-session-banner-exchange-invalid-format)
banner exchange: could not read protocol version - given the message & what I could find about this from the internet, my guess is that invalid security protocols between the client and server are used.
Connection closed by <ip_addr> port <port_number> [non-preauth] - connection without an attempt to authenticate. Typically indicative of SSH port scanning for any known vulnerabilities within the service to exploit (see: https://serverfault.com/questions/1093505/sshd-difference-between-connection-closed-and-disconnected-from-in-lo)
Connection closed by <ip_addr> port <port_number> [preauth] - my guess is invalid SSH keys preventing connection (see: https://serverfault.com/questions/847272/connection-closed-by-ip-preauth)
Connection closed by invalid/authenticating user:

Brute force scenario invoking the following command (taken from: https://superuser.com/questions/1696687/error-authenticating-a-domain-user-for-ssh-connection):

If the log read authenticating instead of invalid, it means the attacker used a valid username. This also rings true for all other invalid/authenticating user failed event logs.
Connection reset by invalid/authenticating user - my guess is that this is a similar scenario as the one directly above; see the time where I tried authenticating as the user jack from earlier
Connection reset by <ip_addr> port <port_number> [non-preauth]:

Server-side issue; my guess is that the resources needed to establish the connection can’t be reached (see: https://github.com/orgs/community/discussions/58249 & https://stackoverflow.com/questions/61185751/ssh-kex-exchange-identification-read-connection-reset-by-peer)
Connection reset by <ip_addr> port <port_number> [preauth] - server couldn’t receive the credentials provided by the source IP address while attempting to authenticate (see: https://stackoverflow.com/questions/68670742/sshd-connection-issue-connection-reset-by-ip-port-x-preauth)
Unable to negotiate with <ip_addr> port <port_number>: no matching key exchange method found. Their offer: <offer> - the client & server don’t share at least one cipher for secure connecting (see: https://unix.stackexchange.com/questions/402746/ssh-unable-to-negotiate-no-matching-key-exchange-method-found)
Disconnected from invalid/authenticating user - similar to the Connection closed by invalid/authenticating user failed event. The only difference between the two lies in how the connection was closed, but the difference is so miniscule that both events may as well be considered the same (refer to the same source mentioned under Connection closed by <ip_addr> port <port_number> [non-preauth] for more details)
Disconnecting invalid/authenticating user: too many authentication failures - as the event states, the attacker was disconnected because they reached the max amount of authentication attempts
For the record, I should note that some of these explanations are incorrect. Notably, the Connection closed/reset events; they’re so remarkably identical to each other that I was having trouble finding resources on the internet that clearly explained how they differentiated from each other, so I conjured up my own conclusions based on the facts that I could gather. I did this question as a way to gather some basic-level insights into attacker behavior & their machine setups.
osTicket Report
After answering all questions to the best of my ability, I reported my findings in the “Attempted SSH Brute Force” ticket:


Home Lab Phase: Changing SSH Failed Authentication Query
Back in the “Creating Reports, Alerts, and Dashboards” section of this part, I mentioned about having to use a not-so-entirely-accurate query for the failed authentication events, due to issues with the query I was intending to use. During my work on Question 4 of the investigation, I came up with queries for each of the various types of failed authentication logs that I uncovered. Here, I had the realization that I could combine all of these queries into one that I’m positive would target every failed authentication event generated by my instance. The queries I constructed are:
index="linux-ssh-events" eventtype=sshd_session_start NOT action=started - for these events:
Connection closed by invalid user
banner exchange: could not read protocol version
banner exchange: Invalid format
Unable to negotiate with <ip_addr> port <port_number>: no matching key exchange method found. Their offer: <offer>
index="linux-ssh-events" eventtype=sshd_session_end name="Connection closed" - for these events:
Connection closed by authenticating user
Connection closed by <ip_addr>
index="linux-ssh-events" ((disconnected from) OR disconnecting) (authenticating OR invalid) user - for these events:
Disconnected from invalid/authenticating user
Disconnecting invalid/authenticating user: too many authentication failures
index="linux-ssh-events" (connection reset by NOT eventtype=nix_errors) - for these events:
Connection reset by invalid/authenticating user
Connection reset by <ip_addr> port <port_number>
Joining all these queries together, my new query for failed SSH authentication events is:
index="linux-ssh-events" AND ((eventtype=sshd_session_start NOT action=started) OR (eventtype=sshd_session_end name="Connection closed") OR (((disconnected from) OR disconnecting) (authenticating OR invalid) user) OR ((connection reset by NOT eventtype=nix_errors)))
I proceeded to make this query change to my SSH failed authentication map & table panels:


And with that, I now have my final dashboard for a quick glance at authentication activity:



