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
With the Linux server all set up and my **Authentication Activity** dashboard updated, I can finally begin the real show of this lab: Actually doing investigative log analysis! I'd like to start things off with investigating the SSH authentication activity. Before getting things rolling, I did a couple things. First, I exposed the Linux server to the internet throughout September 22 to generate additional telemetry. Next, after removing the test tickets in osTicket, I generated two tickets manually by running two `curl` commands in MYDFIR-Splunk’s PowerShell terminal:
![](/screenshots/213.png)
![](/screenshots/214.png)
![](/screenshots/215.png)
![](/screenshots/216.png)

These tickets are where I’ll report the results of my analyses on each type of brute force authentication activity. Lastly, I made some failed SSH connections to the Linux server to generate some extra events, all of which I've never seen before from my telemetry gathering sessions:
![](/screenshots/217.png)
![](/screenshots/218.png)
![](/screenshots/219.png)
![](/screenshots/220.png)

With that done, I had no idea where to start at when it comes to performing a proper investigative process, so I turned to the Day 26 video for advice. I got some good starting questions, including:
  - Any successful attempts? If so, what did the attacker do?
  - Is the IP address known to perform brute force attacks?
  - What usernames did the suspicious IP address target?

From there, I conjured up a question of my own, befitting of the circumstances from using AWS:
  - What happened in the various types of failed login attempts?

### Question 1: Any successful attempts (not from my IP address)? If so, what did the attacker do?
Taking a quick glance at the SSH successful authentications on my dashboard, I do see some successful activity, but all of them came from my laptop’s IP address:
![](/screenshots/221.png)
![](/screenshots/222.png)

For further verification, I ran a query searching for the word “Accepted” in the logs. I ended up getting the same events:
![](/screenshots/223.png)
![](/screenshots/224.png)

So no, there hasn’t been a successful attempt from any outside entity.

### Question 2: What activity did I perform in my first successful authentication session?
While there was no successful attempt that didn't come from me, I did this anyway for the sake of practice. Grabbing the timestamp for the oldest event from my successful authentication report, I ran a search for all events after that time, with the `reverse` command piped in to sort the events from oldest-newest so that I can start the analysis chain from the time of connection for the first session:
![](/screenshots/225.png)
![](/screenshots/226.png)

Next, I began crafting up some filters to exclude events that are unnecessary to my investigation. First, I dealt with the “session opened” and “session closed” events, which comprised the majority of the excess noise:
![](/screenshots/227.png)

Despite the filter, there's still too many logs to analyze in a timely manner. However, considering the amount of successful authentications that came from me, this search not only accounts for my activities from the other login sessions, but also that I had to have disconnected from the Linux server at some point in my first login session, which would’ve been logged. To find this disconnection event, I modified the query to include my IP address. With the modification, I found the event almost immediately:
![](/screenshots/228.png)

Grabbing the event’s timestamp, I re-modified the query, focusing the time range between the time of connection & the time of disconnection for my first login session, as well as removing my IP address:
![](/screenshots/229.png)

Now, only events that occurred during the first login session are returned; the amount returned is feasible enough to work with. Combing through the events, this is what I did in my first login session:
  - Ran the commands `apt-get update` & `apt-get upgrade` (repository updating)
  - Utilized the `dpkg` command to initiate the download of a `splunkforwarder` Debian package. No event of the package itself being downloaded is recorded; this would exist in network packet captures instead.
  - The `nano` command was ran to create a `deploymentclient.conf` file in the `local` directory for `splunkforwarder`, then was subsequently removed
  - `chown` was ran recursively on the `splunkforwarder` directory to change its ownership to `ubuntu`, which is the main username for the Linux server
  - `ufw allow` was ran twice, one to allow traffic through ports 8089 and again for ports 9997 (the default ports used by splunkd and Splunk receivers, respectively)

Much of what happened in my first login session involved setting up the Linux UF, which included tasks like connecting it to the deployment server and preparing it for log forwarding.

### Question 3: Is the IP address known to perform brute force attacks? And what usernames did the suspicious IP address target?
Looking at all failed SSH authentications on my dashboard, there's a lot of IP addresses to choose from. For the sake of time, I’ve chosen to use the one with the most events to answer this question. First, getting the easy stuff right out the gate: A quick glance at the dashboard is all I need to determine the users targeted:
![](/screenshots/230.png)

As for whether the IP address is a known threat, Steven mentions a couple of useful resources in the Day 26 video: [AbuseIPDB](https://www.abuseipdb.com/) and [GreyNoise](https://www.greynoise.io/). Running the IP address through AbuseIPDB, I get my answer almost right away - yes:
![](/screenshots/231.png)
![](/screenshots/232.png)
![](/screenshots/233.png)

For additional intel, I ran the IP through GreyNoise. The site also reveals it to be known to perform brute force attacks:
![](/screenshots/234.png)
![](/screenshots/235.png)

### Question 4: What happened in the various types of failed login attempts? (In other words, explain each type of failed authentication event)
While I'm at it, I've also looked into the various types of failed authentication logs generated by my Linux server, considering the way I authenticate into my Linux instances in AWS. Finding concrete information about each one on the internet proved to be difficult than I expected; some of the failed events were either too similar to each other, had vague information, or I just straight up couldn't find any.

These are the events that I could find definitive conclusions to:
  - `banner exchange: Invalid format` - [scanner](https://security.stackexchange.com/questions/281647/what-are-these-sshd-session-banner-exchange-invalid-format)
  - `Connection closed by <ip_addr> port <port_number>` - connection without an attempt to authenticate. [Typically indicative of SSH port scanning for any known vulnerabilities within the service to exploit](https://serverfault.com/questions/1093505/sshd-difference-between-connection-closed-and-disconnected-from-in-lo)
  - `Unable to negotiate with <ip_addr> port <port_number>: no matching key exchange method found. Their offer: <offer>` - [the client & server don’t share at least one cipher for secure connecting](https://unix.stackexchange.com/questions/402746/ssh-unable-to-negotiate-no-matching-key-exchange-method-found)
  - `Connection closed by invalid/authenticating user` & `Connection reset by invalid/authenticating user` - invalid credentials provided; `invalid` user means the incorrect username was used, while `authenticating` user means the correct username was used
    ![](/screenshots/236.png)
    - Finding conclusive information on the differences between `Connection closed` and `Connection reset`, on the other hand, was a different story
  - `Disconnected from invalid/authenticating user` - [same as `Connection closed by invalid/authenticating user` & `Connection reset by invalid/authenticating user`](https://serverfault.com/questions/1093505/sshd-difference-between-connection-closed-and-disconnected-from-in-lo)
    - However, I couldn't find any conclusive information on the differences between `Disconnected` and `Connection closed/reset`
  - `Disconnecting invalid/authenticating user: too many authentication failures` - as the description suggests, the attacker was disconnected because they reached the max amount of authentication attempts

And these are the rest, all of which I could find vague to no conclusive information on:
  - `banner exchange: could not read protocol version`
  - `Connection closed by <ip_addr> port <port_number> [preauth]`
  - `Connection reset by <ip_addr> port <port_number> [preauth]` - [server couldn’t receive the credentials provided by the source machine while attempting to authenticate](https://stackoverflow.com/questions/68670742/sshd-connection-issue-connection-reset-by-ip-port-x-preauth)
  - `Connection reset by <ip_addr> port <port_number>`
    ![](/screenshots/237.png)

### osTicket Report
After answering all questions to the best of my ability, I reported my findings in the “Attempted SSH Brute Force” ticket:
![](/screenshots/238.png)
![](/screenshots/239.png)

## Changing SSH Failed Authentication Query
During Question 4 of the investigation, I came up with queries for finding all the various types of failed authentication events generated by my Linux server. I had the realization here that I could combine all these queries into one that would target every failed authentication event generated. The queries I constructed are:
- `index="linux-ssh-events" eventtype=sshd_session_start NOT action=started` for these events:
  - Connection closed by invalid user
  - banner exchange: could not read protocol version
  - banner exchange: Invalid format
  - Unable to negotiate with \<ip_addr\> port \<port_number\>: no matching key exchange method found. Their offer: \<offer\>
- `index="linux-ssh-events" eventtype=sshd_session_end name="Connection closed"` for these events:
  - Connection closed by authenticating user
  - Connection closed by \<ip_addr\>
- `index="linux-ssh-events" ((disconnected from) OR disconnecting) (authenticating OR invalid) user` for these events:
  - Disconnected from invalid/authenticating user
  - Disconnecting invalid/authenticating user: too many authentication failures
- `index="linux-ssh-events" (connection reset by NOT eventtype=nix_errors)` for these events:
  - Connection reset by invalid/authenticating user
  - Connection reset by \<ip_addr\> port \<port_number\>

Joining all these queries together, my new query for failed SSH authentication events is:
`index="linux-ssh-events" AND ((eventtype=sshd_session_start NOT action=started) OR (eventtype=sshd_session_end name="Connection closed") OR (((disconnected from) OR disconnecting) (authenticating OR invalid) user) OR ((connection reset by NOT eventtype=nix_errors)))`

Modifying the queries in my SSH failed authentication map & table panels:
![](/screenshots/240.png)
![](/screenshots/241.png)

And with that, this is the final version of my **Authentication Activity** dashboard:
![](/screenshots/242.png)
![](/screenshots/243.png)
![](/screenshots/244.png)

