# Part 12: What EDR to use? (Day 29)
Throughout this lab, I've gone through so much, from setting up Splunk to investigating two types of authentication activity and a malware situation, to establishing a ticketing system. Now, to cap off the main challenge, I’ve decided to make an attempt on integrating an endpoint detection & response (EDR) tool with Splunk. With Elastic Defend not being a viable option to begin with, I did some prior research for other EDR tools that could work with Splunk, preferably free of charge. This led me to this [YouTube short by MyDFIR](https://www.youtube.com/shorts/BzRXa4IGOy8), where Steven mentions an EDR called Aurora Lite. Looking up this tool, I found it to fulfill much of my needs; the only catch was that this is a watered down version of Aurora (the full version), but Steven echos some wise words of wisdom in the same short:

> ***Tools are just tools. What matters most is knowing how to use them to make sense of data and to tell a story.***

What I took away from this is that even if it’s a different experience from Elastic Defend, what's most important for me at the moment is understanding the fundamentals, or concepts, of EDRs. As such, I’ve settled upon Aurora Lite for my lab’s EDR.

## Installing Aurora Lite
1. First, I downloaded Aurora Lite by heading to the [Aurora site](https://www.nextron-systems.com/aurora/), then clicked “Download” > “Download Aurora Lite”.
2. After entering in my name & email, I checked the second box (the required one that sends a verification), then clicked “Submit”.
3. In the inbox for the email used in the previous step, I opened the confirmation email sent by Nextron Systems and clicked Confirm. Another email from Nextron appeared shortly after, which contains two download links: The application license, and the application itself. I want to download both onto the Windows server, not my host.
4. After connecting to the Windows server, I right-clicked the license download link and selected "Copy Link". Then, on the Windows server, I opened up Microsoft Edge and pasted the copied link there.
5. I repeated the previous step for the application download link. A zip file will be provided. I also kept both download pages up.
6. To be safe, I verified the SHA1 hash of the file first before unzipping:
    1. With PowerShell opened, I ran the command `Get-FileHash -Path <file_path_to_zip_file> -Algorithm SHA1`. This produces a hash.
    2. With the hash copied, I returned to the application download page, which provides the original SHA1 hash of the zip file.
    3. To compare the hashes, I opened Finder by pressing CTRL+F, then pasted the hash from PowerShell into the “Find…” text box. If a match is found (the hash on the page is highlighted), that means the file is legitimate and therefore safe to proceed. If not, then the download was compromised; I must delete the file and restart the process.
7. After verifying that the file is legitimate, I proceeded to install Aurora Lite by following the “[Quick Installation](https://aurora-agent-manual.nextron-systems.com/en/latest/usage/installation.html#quick-installation)” section of the Aurora agent documentation, up to step 5.
8. For step 5, I want to install with the standard configuration preset for detections. To do so, I ran the command `aurora-agent.exe --install -c agent-config-standard.yml`.
9. Afterwards, I proceeded to finish the installation; I’m able to verify new events involving Aurora from the Windows event logs:
![](/screenshots/289.png)
Conveniently, Aurora writes its logs to the Application event logs by default, which are already set up to be ingested into Splunk:
![](/screenshots/290.png)

## Testing Aurora Lite
Now that Aurora Lite is installed onto the Windows server and integrated with Splunk (via log ingestion), I want to test Aurora to see if the tool will work out for me. For that, I’m bringing back in Mythic; I want Aurora Lite to detect and remove the Apollo agent from the machine. Before beginning, I modified my malware alert to push tickets to osTicket:
![](/screenshots/291.png)
![](/screenshots/292.png)
Body:
![](/screenshots/293.png)

With that out of the way, I redownloaded the Apollo agent onto the Windows server, hoping that Aurora Lite, out of the box, detects the malicious binary being downloaded and responds by automatically removing it from the server. The download does trigger Aurora’s detection, as evident in the logs:
![](/screenshots/294.png)
![](/screenshots/295.png)

But no removal response occurred:
![](/screenshots/296.png)

I ran the binary, hoping that would get Aurora to remove it, or at the very least, stop the binary from performing its malicious activities. Unfortunately, the outcome here was similar; Aurora detected the binary running:
![](/screenshots/297.png)

But no response was executed and the C2 was established, indicated by the last check in time of the selected interaction being mere seconds ago:
![](/screenshots/298.png)

With Aurora Lite failing to execute the response I wanted in both cases, I spent some time troubleshooting by reading more of the documentation. I ended up checking the Aurora service’s status by running the command `aurora-agent-64.exe --status`. Viewing the resulting report, I noticed that response actions were disabled, hence the lack of responses:
![](/screenshots/299.png)

### Building Custom Sigma Rules & Responses
Before proceeding to enable these actions, I wanted to first build a dedicated sigma rule that detects the downloading of an Apollo binary onto any folder, not exclusively the `Public` directory, and removes the binary as a response action to mitigate Apollo C2 attacks, in addition to giving me a crash course on detection engineering and the fundamental tasks involved.

To get started with things, I referred back to the Aurora documentation, reading through [the chapter on creating responses](https://aurora-agent-manual.nextron-systems.com/en/latest/usage/responses.html), as well as [the chapter on custom signatures and IoCs](https://aurora-agent-manual.nextron-systems.com/en/latest/usage/custom-signatures.html). From there, I looked at examples across the readings, Aurora Lite's base rules under `signatures/sigma-rules`, and Aurora's `response-sets` directory, which not only helped provide clarity on knowledge gaps such as response options, but also served as helpful templates for my own rules; Section 10 provides a link to the [sigma specification](https://github.com/SigmaHQ/sigma-specification?tab=readme-ov-file), which includes the full rule structure and the full list of options that can be specified.

--There, I found that not only can I add my custom rules under the  directory--

Once I've gotten a grasp on everything I need to know about sigma rules and responses, I went to work on crafting my rules. In addition to the downloaded Apollo binary detection rule, I want another rule that detects the malicious binary being executed and responds by killing the process, then removing said binary. This rule will act as a safeguard should the download rule fail. I constructed this rule first, which looks like this:
![](/screenshots/300.png)

Here, the rule works by monitoring the Sysmon events `logsource` under `Applications and Services Logs/Microsoft/Windows` in Event Viewer for any event with EventID=1 and OriginalFileName=`Apollo.exe` (`detection`). If such an event is found, the rule sounds the alarm and executes its `response`, first killing the process ID assigned to the executable, then removing the executable from the server, in order.

Next, I constructed the downloaded Apollo binaries rule. I went a different route here, and used the following base rule in a response set:
![](/screenshots/301.png)
![](/screenshots/302.png)

My original idea here was to have the rule monitor the Sysmon events for any event with EventID=11. However, when taking a look at the events with this ID in Splunk, I found out that they don’t contain the `OriginalFileName` field, which complicated matters. For the sake of simplicity, I ended up utilizing the base rule above, despite the goals I've outlined in this section. Additionally, I originally wanted both the **Apollo C2 Malware Executed** and **Suspicious Binaries and Scripts in Public Folder** rules utilizing the same response. However, without a common field containing the executable's absolute path being shared between the rules, I wasn't able to implement this, so I gave each rule their own responses as shown.

Regarding my downloaded binary rule, I only realized when writing my first draft of this part that the rule only targets a specific directory, not all, which is what I was supposed to do. But that didn’t matter, because shortly after constructing my rules and responses, I was faced with the reality that I can’t use my custom responses; rereading through the documentation, it says that Aurora’s Lite version only allows the usage of predefined ones. The full list of predefined responses are as follows:

Adding my custom rules & responses to the `custom-signatures/sigma-rules` directory and testing them out, I didn't get the results I was expecting.

Due to this restriction, I was forced to remove all custom response actions from my rules. Since my response set for the “Suspicious Binaries and Scripts in Public Folder” rule only has the custom response, it ended up going unused as a result. That leaves my “Apollo C2 Malware Executed” rule as the only custom rule left, which now looks like this:

Making Changes to the Installation
Having now established my custom rule, pushing the change is a different story. When installing Aurora Lite as a service, the service will operate based on the flags specified in the command line, as well as the files that were in the temporary folder at the time of installation. At the time I installed Aurora, I didn’t start up any custom rules and responses yet, meaning that my current installation is running without any of them active. Anytime I want to push changes to Aurora post-installation, such as making it use custom rules and activating responses, I’ll have to reinstall the service. Before doing so, first I need to add my rule to the appropriate directory in the temporary folder that I used when I first installed Aurora Lite (which in this case was C:/aurora):

Although, I’ve also added the rule under the installation:

Then, I reran the install command under the temp folder with the following flags:

After doing the re-installation, Aurora Lite is now able to execute response actions and should actively be using the “Apollo C2 Malware Executed” rule. To test the rule & response out, I went and got my Apollo binary up and running. The first time executing it, the response failed to fire off, which at the very least indicates that the detection rule is indeed working. Checking the corresponding error log, I saw the problem right away: I was using the wrong process ID field:


After re-installing the service again to echo the change, I ran the binary again for another test. The test failed again. Checking the log, this time the error was caused by the fact that the binary was run with escalated privileges (a.k.a. as root):

To fix this, I added the flag lowprivonly: false under response: of my rule. Then I ran the binary for a third time; maybe the third time’s the charm. Indeed it was, because this test succeeded:

The corresponding log in Splunk:



