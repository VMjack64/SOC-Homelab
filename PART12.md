Previous Part                                     | Return to Introduction                  | Next Part
------------------------------------------------- | --------------------------------------- | ---------
[Part 11: The RDP Investigation Part](/PART11.md) | [Introduction](/README.md#introduction) | [Part 13: The Bonus Challenge](/PART13BONUS.md)

[Part 12: What EDR to Use?](#part-12-what-edr-to-use-day-29)
- [Installing Aurora Lite](#installing-aurora-lite)
- [Testing Aurora Lite](#testing-aurora-lite)
    - [Building Custom Sigma Rules & Responses](#building-custom-sigma-rules--responses)
    - [Making Changes to the Installation](#making-changes-to-the-installation)

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

Once I've gotten a grasp on everything I need to know about sigma rules and responses, I went to work on crafting my rules. In addition to the downloaded Apollo binary detection rule, I want another rule that detects the malicious binary being executed and responds by killing the process, then removing said binary. This rule will act as a safeguard should the download rule fail. I constructed this rule first, which looks like this:
![](/screenshots/300.png)

Here, the rule works by monitoring the Sysmon events `logsource` under `Applications and Services Logs/Microsoft/Windows` in Event Viewer for any event with EventID=1 and OriginalFileName=`Apollo.exe` (`detection`). If such an event is found, the rule sounds the alarm and executes its `response`, first killing the process ID assigned to the executable, then removing the executable from the server, in order.

Next, I constructed the downloaded Apollo binaries rule. I went a different route here, and used the following base rule in a response set:
![](/screenshots/301.png)
![](/screenshots/302.png)

My original idea here was to have the rule monitor the Sysmon events for any event with EventID=11. However, when taking a look at the events with this ID in Splunk, I found out that they don’t contain the `OriginalFileName` field, which complicated matters. For the sake of simplicity, I ended up utilizing the base rule above, despite the goals I've outlined in this section. Additionally, I originally wanted both the **Apollo C2 Malware Executed** and **Suspicious Binaries and Scripts in Public Folder** rules utilizing the same response. However, since both don't share a `Image` or `TargetFileName` field containing the executable's absolute path, I wasn't able to implement this, so I gave each rule their own responses as shown above.

Despite going against my goals regarding the downloaded Apollo binaries rule, it ultimately didn’t matter, because shortly after constructing my rules and responses, I read through the feature list available in Aurora Lite, and realized that I misinterpreted the clause about custom responses; I can't use them, and am limited to predefined responses. The full list of predefined responses are as follows:
![](/screenshots/303.png)

Due to this restriction, I was forced to remove all custom response actions from my rules. This left my downloaded Apollo binaries rule unused as a result, leaving only my **Apollo C2 Malware Executed** rule, which now looks like this:
![](/screenshots/304.png)

### Making Changes to the Installation
Thankfully, despite the setback, I still have my Apollo executed rule that I can use. Getting Aurora to use the rule, on the other hand, was a different story. When I installed Aurora Lite as a service, the service operates based on the flags I specified in the command line, in addition to the files that were present in the temporary folder at the time of installation. Back then, I didn’t have any custom rules & responses and I didn't specify the flag to activate responses, meaning that my current installation is running without response actions and my Apollo executed rule active. If I want Aurora Lite to use my custom rule (as well as any other changes) post-installation, I'll need to reinstall the service, after adding my rule to the `custom-signatures/sigma-rules` directory under the temporary folder that I used for the installation process (which in my case was `C:/aurora`). Just in case, I’ve also added my rule under the current installation's folder:
![](/screenshots/305.png)

Once I've added my rules, I reran the install command, this time with the `--activate-responses` flag to tell the re-installation to use response actions:

![](/screenshots/306.png)

Now, the new Aurora Lite service is able to execute responses and actively using my **Apollo C2 Malware Executed** rule. To test the rule & response out, I ran the Apollo binary. The first time executing it, the response failed to fire off. Looking into the Aurora logs, I found the error corresponding to this event, and saw the problem right away: I was using the wrong process ID field:
![](/screenshots/307.png)

At the very least, this log confirms that the detection rule is working. I corrected the field name:
![](/screenshots/308.png)

Then re-installed the service to echo the change. Afterwards, I ran the binary again for another test. The test failed again. Checking the logs, this time the error was caused by the fact that the binary was run with escalated privileges (a.k.a. as root):
![](/screenshots/309.png)

To fix this, I added the flag `lowprivonly: false` to my rule's response. After echoing the change, I ran the binary for a third time, and this time the test succeeded:
![](/screenshots/310.png)
![](/screenshots/311.png)
![](/screenshots/312.png)

Previous Part                                     | Return to Introduction                  | Next Part
------------------------------------------------- | --------------------------------------- | ---------
[Part 11: The RDP Investigation Part](/PART11.md) | [Introduction](/README.md#introduction) | [Part 13: The Bonus Challenge](/PART13BONUS.md)
