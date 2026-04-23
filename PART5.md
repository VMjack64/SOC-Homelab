Previous Part                                    | Return to Introduction                  | Next Part
------------------------------------------------ | --------------------------------------- | ---------
[Part 4: Creating the Windows Server](/PART4.md) | [Introduction](/README.md#introduction) | [Part 6: Getting Logs Ingested Into Splunk](/PART6.md)

[Part 5: Prepare for Log Ingestion](#part-5-prepare-for-log-ingestion-days-7-8-9)
- [Deployment Server Creation & Setup](#deployment-server-creation--setup)
- [Installing Windows Universal Forwarder (UF)](#installing-windows-universal-forwarder-uf)
- [Windows Server Setup](#windows-server-setup)
- [Creating Indexes & Setting up Sysmon Parsing](#creating-indexes--setting-up-sysmon-parsing)

# Part 5: Prepare for Log Ingestion (Days 7, 8, 9)
Once the Windows server generated enough telemetry, I began the process of prepping the server for ingesting logs into Splunk. My tasks here are to install the universal forwarder onto the Windows server, as well as create the deployment server. Before starting, I turned off the server for a moment to add the `Windows-Server-Firewall` to the instance’s security group list, under **Change security groups**. However, I didn’t remove the `unsecure-firewall` from the list, a decision that I would later find out to be a huge mistake. 

## Deployment Server Creation & Setup
The first task I did was create the deployment server, which will make setting up the universal forwarder easier later on. Referring to the [key pointers on deployment servers back in Part 2](/PART2.md#key-pointers-on-deployment-servers), I want a dedicated instance with the following specifications:
  - Name: MYDFIR-Splunk-Deployment-Server
  - AMI: Ubuntu, Ubuntu Server 24.04 LTS
  - Architecture: 64-bit (x86)
  - Instance type: t3.small. Despite the resource usage of deployment servers, I didn't feel like this server needed the same processing power as MYDFIR-Splunk.
  - Key pair: RSA type, .pem format
  - Network settings:
    - VPC: MYDFIR-30Day-SOC-Challenge
    - Subnet: splunk-subnet
    - Auto-assign public IP: Enabled
    - Firewall (security groups): Select existing security group; choose `30-Day-MYDFIR-SOC-Challenge` in the dropdown under **Common security groups**
  - Configure storage: 1 volume only:
    - Text box: 30 GiB
    - Dropdown box: Left as whatever option is selected

After creating the instance, I modified the outbound rules for the `unsecure-acl` NACL to add in the following:
![](/screenshots/26.png)

Where `172.20.0.30/32` is the private IPv4 address of MYDFIR-Splunk while `172.20.0.21/32` is the private IPv4 address of MYDFIR-Splunk-Deployment-Server.

Then, I navigated to the **Security groups** screen to edit the inbound rules for the `30-Day-MYDFIR-SOC-Challenge` firewall, with the following added:
![](/screenshots/27.png)
![](/screenshots/28.png)
![](/screenshots/29.png)

Where `172.20.0.214/32` is the private IPv4 address of the Windows server instance.

> [!NOTE]
> After adding and/or modifying rules on any NACLs and/or firewalls, it’s a good idea to reboot all active instance(s) using them to ensure the updated rules take effect. Restart the instance(s) through AWS, not in the instance itself.

Once the new rules have been added, I now have to install Splunk Enterprise, which will have deployment server functionality active by default. For this step, I pretty much just repeated everything under the [Splunk Server Setup](/PART3.md#splunk-server-setup) section to install Splunk Enterprise onto MYDFIR-Splunk-Deployment-Server. The only difference is that on step 6, I ran the command `sudo ufw allow 8089` in addition to the other two `ufw allow` commands.

## Installing Windows Universal Forwarder (UF)
Now that the deployment server is ready to go, it just needs a Splunk instance to start managing. For that, I began the second task of installing the Splunk UF onto the Windows server. After turning the Windows server on, I navigated to the **Connect** screen for it and followed the on-screen prompts to connect to the server via RDP with my laptop's built-in Remote Desktop Connection application. 

After establishing a connection to the server, I opened up a browser on my host machine and went to the [official Splunk website](https://www.splunk.com/) to download the installer for the UF.
  - On the “Choose Your Download” screen for the universal forwarder, on the "Windows" tab, click “Copy wget link” next to the “Windows Server” option.

1. Back in the Windows server remote desktop session, I opened up a PowerShell terminal, then pasted & entered in the wget command to download the `.msi` installer.
2. After locating the installer under `C:/Users/<username>`, I ran it, then went through the installation process:
    - Accepted the license agreement, and made sure on-premises is selected. Then I clicked “Next”.
      - Under “Customize Options” are the advanced settings, which are more tailored for a business setting. As this is a personal project, I went the simple route instead and skipped these settings.
    - I created a username & password that the forwarder will use for authentication.
    - On the **Deployment Server** screen, I entered in the private IPv4 address of the deployment server in the left textbox, while in the right textbox (port for connecting), I entered in 8089.
    - On the **Receiving Indexer** screen, I entered in the private IPv4 address of MYDFIR-Splunk in the left textbox, while I specified 9997 in the right textbox.
    - Finally, I clicked “Install” to install the forwarder onto the Windows instance. Conveniently, the Splunk UF for Windows is configured by default to automatically start upon booting up the instance.
> [!NOTE]
> As a good security measure, it’s recommended to create a username & password that’s completely different from the one used for the Splunk and deployment servers.

## Windows Server Setup
After installing the UF on Windows, I restarted the deployment server (by rebooting through AWS or running the command `./splunk restart` in its SSH terminal session) so that it can receive the connection from the Windows UF. Then, I accessed the Splunk web GUI for the deployment server, and navigated to “Settings” > “Agent management”, where I can see the Windows instance show up if the connection is successful:
![](/screenshots/30.png)

With the Windows server connected, I did additional setup tasks on the server, most notably installing Sysmon. The tool logs additional information not present in the base Windows logs, such as process hashes, process GUIDs, executed commands, DLLs, and source & destination network connections. These make it possible to spot the presence of malware and more subtle malicious activity. For the installation process, I just followed Day 9 of Steven’s series.

> [!NOTE]
> Other Sysmon configuration files exist outside of Olaf’s config. One such is [SwiftOnSecurity’s configuration file](https://github.com/SwiftOnSecurity/sysmon-config). Consider researching all the different config files and figure out which one fits best with what you want out of Sysmon. Or, if you're very knowledgeable, conjure up your own config.

## Creating Indexes & Setting up Sysmon Parsing
The last additional task I did before proceeding was creating 3 new indexes for MYDFIR-Splunk: `windows-server-events`, `windows-sysmon-events`, and `windefender-events`. By default, the UF will forward all logs to the `main` index, but the Splunk documentation states that it's best practice to forward the logs to a dedicated index for a few reasons. One in particular, is because it can speed up the search process; some unnecessary logs in the `main` index have a significant appearance rate than others, so searching one of the dedicated indexes means Splunk doesn't have to spend time filtering out those logs. I navigated to "Settings" > "Indexes" in MyDFIR-Splunk's web GUI and created the indexes from there.
  - During the creation process, there were a few additional options available for each index; for the purposes of this lab, I left them as is.

With the indexes made, I also set up MYDFIR-Splunk to be able to parse Sysmon logs. This is something I recalled from doing MyDFIR's [Basic Home Lab series](https://www.youtube.com/watch?v=-8X7Ay4YCoA&pp=ygUVbXlkZmlyIGJhc2ljIGhvbWUgbGFi) prior: Splunk doesn’t automatically parse Sysmon logs by default. To allow this, I need to install the ***Splunk Add-on for Sysmon*** from the Splunk add-on store, accessible by clicking on “Apps” > “Find More Apps”.

Previous Part                                    | Return to Introduction                  | Next Part
------------------------------------------------ | --------------------------------------- | ---------
[Part 4: Creating the Windows Server](/PART4.md) | [Introduction](/README.md#introduction) | [Part 6: Getting Logs Ingested Into Splunk](/PART6.md)
