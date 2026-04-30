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
I got to work on installing the UF onto the Linux instance. The installation process is similar to the step-by-step process of installing Splunk Enterprise onto an Ubuntu instance from the “Splunk Server Setup” subsection, but with a few changes:
3b) Click under “Universal Forwarder” instead of “Splunk Enterprise”.
3d) Under the “Linux” tab, use the 64-bit .deb wget link.
5) The Linux forwarder will be installed under ‘/opt/splunkforwarder’ by default.
7) I ran into a problem here where the Splunk forwarder wouldn’t automatically run under systemd without root privileges. Looking up the Splunk documentation, a solution is provided requiring Polkit rules:

Conveniently, the Polkit library happened to be pre-installed onto the Linux instance:

After following the documentation closely, the problem was successfully remedied.
8b) Allow port 8089 instead of 8000.
10) Ignore; Splunk forwarders have no web GUI.
Home Lab Phase: Adding Linux UF to the Deployment Server and Connecting to the Splunk Indexer
With the UF installed & running, I want to add the UF to the deployment server, as well as point it to the indexer in Splunk to begin forwarding logs:
In the CLI for the Linux SSH instance, I created a deploymentclient.conf file under /opt/splunkforwarder/etc/system/local.
Under the deploymentclient.conf file, I added the following stanza:
	[deployment-client]

[target-broker:deploymentServer]
# Specify the deployment server; for example, "10.1.2.4:8089".
targetUri= <URI:port>
	Where <URI:port> = <private-ip-address-of-deployment-server-instance>:8089.
Restart the forwarder. If successful, the Linux server should appear in the deployment server’s agent management screen.
After successfully adding the Linux UF to the deployment server, now I need to connect the UF to the Splunk indexer. To do so for a Linux UF, I created (or edited) the outputs.conf file under /opt/splunkforwarder/etc/system/local to add the following stanzas:

Based on what I’ve read in the documentation, the defaultGroup attribute here is used for grouping a set of log types to forward to a specific indexer (though this might be wrong, so don’t quote me on this). But in this case, I’ll just have everything routed to the one indexer over at “MYDFIR-Splunk”.
Given the setup above, the Linux server logs will be forwarded to the indexer’s “main” index upon restarting the forwarder, but just like with the Windows server logs, I want to organize the logs & send them to a custom index instead. To do that, a inputs.conf file with the following stanza is needed:
	[monitor://<absolute_path_to_auth_logs>]
disabled = 0
index = <name_of_custom_index>
Where <absolute_path_to_auth_logs> is /var/log/auth.log (but could be /var/log/secure instead, depending on the type of Linux installation) and <name_of_custom_index> = “linux-ssh-events” (I’ve also made sure to create this index in the main Splunk instance). While I could manually add the stanza into the inputs.conf file under the Linux UF’s etc/system/local directory, I instead used my deployment server to push this configuration (I could’ve done the same for the outputs.conf file, but chose not to in that case). To do so, I pretty much followed the same steps as how I set up the “windows-sysmon-events” (& “windefender-events”) index from the “Stanza Creation” section, creating an app directory called linuxssh-event-logs and configuring a new server class to associate the Linux UF & the app with. With the stanza inputted, I reloaded the deployment server to echo the changes to the Linux UF and begin ingesting Linux server logs to Splunk.
Home Lab Phase: A Deja Vu Moment
After the Linux server’s logs are successfully being ingested into Splunk, I checked out the logs… only to spot a problem almost immediately:



The problem here doesn’t lie within the logs, but in the “Interesting Fields” column: There’s no username or IP address field. For some reason, Splunk has a problem with extracting these fields from the events naturally. To resolve this problem, the first thing that came to mind was using Splunk’s Field Extractor utility, under “Extract New Fields”. After going through the events & looking for patterns, I used the RegEx operation to perform the extraction. Initially, things were going fine, having made 100% accurate extractions for source IP address:

And connection events (invalid or accepted; tried using \b initially, but Splunk has a problem parsing that, so I used \s instead):

But then I got to the usernames. The extraction looked like this:

Unfortunately, it was producing inaccurate results; missing a few events / grabbing a non-username. This could potentially pose a problem with “shadow attacks” in an actual environment, where a suspicious event may be missed during log searching, and if this suspicious event is actually a real threat that successfully gets into the company’s internal network, it’d make things worse. Therefore, this is why I want every event parsed correctly.
To rectify the username issue, I turned to another solution: Installing the “Splunk Add-on for Unix and Linux”. As a prerequisite, I edited the inputs.conf file in the linuxssh-event-logs app directory in the deployment server CLI to add sourcetype = linux_secure (the alternate name for Linux logs) under the stanza.
After completing the prerequisites and reloading the deployment server to echo the changes, I went to Splunk’s web GUI to install the add-on from the add-on store.
Though the add-on is installed, it doesn’t parse the authentication logs located under /var/log by default. I’ll need to activate it by navigating to “Apps” > “Manage Apps”, locating the add-on, then clicking “Set Up”. From there, enable /var/log, then “Save”.
For best practice, I need to disable app visibility for the add-on by clicking “Edit properties” next to the add-on, then set “Visible” to “No”. This essentially makes the add-on invisible to other Splunk apps.
After saving the changes, I re-checked the logs, and see that not only has the sourcetype changed, but I also get some new fields, indicating the solution is a success:



Note that any changes made to the sourcetype only apply to newly ingested logs, not post-ingested ones, just like what happened when changing the hostname of the Windows server that appears in Splunk. I could leave these logs as-is, but in doing so, the Splunk Linux add-on won’t be able to parse the post-ingested logs. Therefore, it’s necessary for the sourcetype of the post-ingested logs to be up-to-date in this case. Looking up on how to accomplish this, my objective here is to re-index these logs, with the updated inputs.conf stanza configurations already in place. I did a couple of preparations beforehand: First, I went to the inputs.conf file and added a [default] stanza to change the hostname of the Linux server that will appear in Splunk, similar to how I did it for the Windows server. Then, I want to backup the “linux-ssh-events” index, just in case something went wrong during the re-indexing process. With help from this community forum, I pulled this off by simply copying the index’s database folder, /opt/splunk/var/lib/splunk/linux-ssh-events, to the home directory.
With preparations complete, I began the re-indexing process. For my first move, after navigating to the “MYDFIR-Splunk” instance CLI to stop Splunk, I ran the command ./splunk clean eventdata -index linux-ssh-events to remove all data from the index. Afterwards, I started up Splunk again. Then, following the guidance of the top post from this community forum, I removed the fishbucket index from the Linux UF via the CLI and restarted the UF to try to tell it to re-ingest the logs. I had assumed the removed logs from the “linux-ssh-events” index still existed in the Linux server’s auth.log file, so I was caught off guard when I checked the Linux index in Splunk and only saw a handful of events. When I went to search the auth.log file for the old logs, I realized that they were erased from the file. I don’t know how this could’ve happened, but my theory is that it’s probably either an automatic system process from turning off the Linux instance or a side effect from executing the command above.
Although the first solution failed & the old logs were erased from the .log file, I still have the backup for the index to fall back on, so I pressed on and searched the internet for a solution that utilizes this backup. Not long after, I found a promising one, in the form of this AI Overview from the following Google search query:

I want the old logs to be re-indexed into the “linux-ssh-events” index, but at the same time, I don’t want to delete any newly ingested data from the index. The solution here is pretty simple; I first copied the index backup into a new index called “messed-up-linux-events”. Then, I performed step 1, running the command ./splunk search "index=messed-up-linux-events sourcetype=auth" -output csv -maxout 0 > ~/source-type-auth.csv in “MYDFIR-Splunk”’s CLI to export the old logs into a .csv file. By default, the “Time range” for the .csv file is “All Time”, according to the Splunk documentation. Once the export is finished, I performed step 3 by running the command ./splunk add oneshot ~/source-type-auth.csv -sourcetype linux_secure -index linux-ssh-events -host "Ubuntu Server 24.04", still in “MYDFIR-Splunk”’s CLI. After following through the overview, I re-checked the Linux index in Splunk to see if the process is a success, which for the most part, yes:




NOTE: Looking back on this, I realized I probably could’ve skipped the entire re-indexing process and instead specified a new sourcetype under “Settings” > “Source Types” named auth with the linux_secure typing. Had I figured this out earlier, this would’ve made my life a lot easier, but I was already too neck deep into the task by then.
Home Lab Phase: Creating Reports, Alerts, and Dashboards
After going through the hassle of setting up the Splunk Linux add-on, all the Linux server logs are now being parsed and providing additional fields with more detailed information. With this, I can finally start building reports, an alert, and a dashboard for Linux SSH authentication events. After familiarizing myself with the available fields for the “linux-ssh-events” index logs, these are the fields/queries that I used to fulfill the following:
Username: user
Source IP address: src
Successful authentication: vendor_action=Accepted
Failed authentication: tag=start action!=started. The most optimal field for this would’ve been action=blocked, but attempting to search this yields no results:

When referring to the Splunk hierarchy back in Part 2 of this lab, theoretically speaking the problem doesn’t seem to lie within the parsing/indexing phase, otherwise all of the events would not have an action field to begin with. Additionally, there’s the fact that some searches (such as the one from when I was checking the results of the re-indexing process) do return events with the action=blocked label. So, this is very likely a problem within the search-time process of Splunk (specifically, the part between the indexing and searching phase). As action=blocked breaks the search process, I had to come up with an alternative, hence tag=start action!=started. And even then, I feel as if this doesn’t return all failed authentication events. I would eventually conjure up a better query, but for now this is the one that seemed the most accurate amongst the other queries that I’ve tried.
Report Query and Alert (Failed Authentications Only)





Body (HTTP Alert Action):

I had tried many ways to get newline formatting to show up in the ticket, but none of them stuck. I thought I was getting somewhere via encodings, considering the ticket actually rendered &lt;br&gt;, but when trying the encoding for newline (%0D%0A), osTicket didn’t render it. With all possible options exhausted, I had no choice but to go with a separator for each field. I was going to use |||, but osTicket wouldn’t render it either, much to my annoyance, so I just went with a bunch of spaces, as shown.
Map (Failed Authentications)

Saved to the “Authentication Activity” dashboard with the title “SSH Failed Authentication Activity Map”
Map (Successful Authentications)

Saved to the “Authentication Activity” dashboard with the title “SSH Successful Authentication Activity Map”
Table (Failed Authentication Activity)


Regarding the empty user column for the United States, India, and Russia, those refer to these types of events:


As well as some events where the user   was specified.
Table (Successful Authentication Activity)

Both tables are saved to the “Authentication Activity” dashboard with the titles “SSH Failed Authentication Activity Table” and “SSH Successful Authentication Activity Table”, respectively
Adding Event Logs to Dashboard
After adding in the SSH authentication panels, I could’ve finalized the dashboard right here, but didn’t because I worry that the blank user slots mentioned just recently might cause some error in judgement; one may mistake them as no user being specified, resulting in potential false negatives. While a few events don’t in fact specify a username, some of the other events actually specify   for the user. As a remedy, I’ve opted to add the failed & successful SSH authentication logs to the dashboard. I was going to use the reports I just created, but because they have their own dedicated time pickers, Splunk wouldn’t allow me to insert them into the dashboard as panels. So, I resorted to the following alternate solution:
In the “Authentication Activity” dashboard, click Edit, then Add Panel.
In the Add Panel menu, click New, then Events.
I entered in the following information, then clicked Add to Dashboard:

Then, I repeated steps 1-3 for successful authentication logs:

Click Save to save the changes.
With the panels created, this is what my dashboard looks like now:






Home Lab Phase: It’s Investigating Time!
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



