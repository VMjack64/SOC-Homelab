# Part 12: What EDR to use? (Day 29)
Throughout this lab, I've gone through a lot, from setting up Splunk to investigating two types of authentication activity and a malware situation, to establishing a ticketing system. Now, to cap off the main challenge, I’ve decided to attempt integrating an endpoint detection & response (EDR) tool with Splunk. With Elastic Defend not being a viable option to begin with, I did some prior research for other EDR tools that could work with Splunk, free of charge. This led me to across one of MyDFIR’s YouTube Shorts, where Steven talks about various tools to learn as a SOC analyst. One of the tools he mentioned was an EDR called Aurora Lite. Looking up this tool, I found it pretty much fulfills my needs; the only catch was that this is a watered down version of Aurora (the full version), but Steven echoed some wise words of wisdom in the same short:
“Tools are just tools. What matters most is knowing how to use them to make sense of data and to tell a story.”
Even if it’s a different experience from Elastic Defend, the most important takeaway I got from this is understanding the fundamentals, or concepts, of the type of tool. With that, I’ve settled upon Aurora Lite for my lab’s EDR tool.
Home Lab Phase: Installing Aurora Lite
After heading to the Aurora site, I clicked “Download”, then “Download Aurora Lite”.
After entering in my name & email, I checked the second box (the required one that sends a verification), then clicked “Submit”.
Navigating to the inbox of the email that I used in step 2, open the confirmation email sent by Nextron Systems and click Confirm. Another email from Nextron should appear shortly, which contains two download links: The application license, and the application itself.
I want to download the license onto the Windows Server instance, not my host computer. To do so, I right-clicked the link, selected “Copy Link”, then opened up Edge on the Windows Server instance and pasted the link over there in the URL. Then, I repeated this step to download the application. Make sure both pages are still up.
For the application, a zip file will be provided. To be safe, I verified the SHA1 hash of the file first before unzipping:
Open PowerShell, then run the command Get-FileHash -Path <file_path_to_zip_file> -Algorithm SHA1.
Copy the hash produced, then return to the application download page. This page provides the original SHA1 hash of the zip file. On the page, press CTRL+F to open Finder, then paste the hash from PowerShell into the “Find…” text box.
If a match is found (the hash on the page is highlighted), that means the file is legitimate and thereby safe to proceed. If not, then the download was compromised; delete the file and restart the download process.
After verifying that the file is legitimate, I proceeded to install Aurora Lite by following the “Quick Installation” section of the Aurora agent documentation, up to step 5.
For step 5 of the documentation, I want to install with the standard configuration preset for detections. To do so, I ran the command aurora-agent.exe --install -c agent-config-standard.yml.
After running the command above, I continued following the last two steps of the documentation; I’m able to verify new events involving Aurora from the Windows event logs:

By default, Aurora writes its logs to the Application event logs, which conveniently are already set up to be ingested into Splunk:

Home Lab Phase: Testing Aurora Lite
After successfully installing Aurora Lite onto the instance and integrating it with Splunk, I want to test and see if it’ll work for me as a free EDR tool. For that, I’m bringing back Mythic; I want Aurora Lite to detect and remove the Apollo agent from the machine. Before starting, I modified my malware alert to push to osTicket, now that the software is integrated with Splunk:


Body:

With those changes made, I went and redownloaded my Apollo agent onto the Windows Server instance, hoping that Aurora Lite, out of the box, detects the malicious binary being downloaded and responds by automatically removing it from the instance. Doing so does trigger Aurora’s detection, as evident in the logs:


But no removal response occurred:

So, the next thing I tried was running the binary, hoping that would get Apollo Lite to remove it, or at the very least, stop the binary from performing its malicious activities. Unfortunately, the same results occurred; Aurora detected the binary running:

But no termination response happened; a connection was established & actively calling back, indicated via the last check in time of the highlighted interaction being mere seconds ago:

Since Aurora Lite failed to initiate a response to the Apollo binary being downloaded and executed, I spent some time viewing more of the documentation to troubleshoot my Aurora service installation. One of the things I ended up doing was checking the service’s status by running the command aurora-agent-64.exe --status. Viewing the resulting status report, I saw that response actions were disabled:

Seeing that response actions are disabled, this means I’ll need to go and enable it for them to work. Before proceeding, however, I’d like to first build a dedicated sigma rule that detects any instance of the Apollo binary within the machine (as in, find the binary in any folder, not exclusively the Public directory) and remove the binary as a response action. Given how easy it is to change the hashes of Apollo binaries, and noting the types of sigma rules that detected the binary when I was reviewing the logs, creating this rule will provide a guaranteed means of mitigating Apollo C2 attacks. When looking back at the tests that I just did, I realized I only scratched the surface of Aurora Lite’s list of detection rules; there might already exist a rule dedicated to detecting Apollo that I wasn’t aware of. Even if that’s the case, this task wouldn’t have been a waste, as it gives me a glimpse into detection engineering and the kinds of tasks involved.
Building Custom Sigma Rules & Responses
To help me get started with building my rules, I referred back to the Aurora documentation. For responses, I read through the chapter on creating responses, which includes examples that helped provide clarity on the options that can be used, as well as showcase the general formatting of response sigmas. For custom rules, I looked at the chapter on custom signatures and IoCs. There, I found that not only can I add my custom rules under the custom-signatures/sigma-rules directory, but the base rules are located under signatures/sigma-rules. Digging into those directories in my Aurora installation provided me with a breadth of examples that helped serve as a template for how I should format my sigma rules:


When viewing the aurora directory, I stumbled across a directory called response-sets, which I also dug into to find more examples of Aurora responses that can serve as inspiration for my own ones:

Additionally, section 10 of the documentation also provides a link to the sigma specification, showing the full structure of a sigma rule, including the full list of options that can be applied.
After taking some time processing all this information, I finally went to work on crafting my rules. As stated, I want Aurora Lite to detect a downloaded Apollo binary & remove it as a response. Additionally, as a safeguard, I want the EDR to detect the malicious binary being executed & respond by killing the process, then removing said binary. For this, I’ll need two rules. My first rule, which focuses on detecting and responding to Apollo executions, looks like this:

Here, the rule monitors the Sysmon log stash, located under “Applications and Services Logs/Microsoft/Windows” in Event Viewer (defined under logsource) for any log with an EventID of 1 and the OriginalFileName being Apollo.exe (defined under detection). If such a log is found, the rule sounds the alarm and should respond accordingly (defined under response). As for my other rule, which will focus on downloaded Apollo binaries, I went a different route; I used the following base sigma rule in a response set:


My original idea for this sigma rule was to have it monitor the Sysmon log stash for any instances of EventID 11. However, when I took a look at the logs with this ID, I found that these logs don’t contain the OriginalFileName field, which really complicated matters. For the sake of simplicity, I ended up deciding to go this route. Additionally, I originally wanted both the “Apollo C2 Malware Executed” and “Suspicious Binaries and Scripts in Public Folder” rules using the same response. However, that would necessitate finding a field containing the absolute path of the executable that happens to be shared between the two rules. I was not able to find such a thing, thus the construction of two separate responses.
Regarding my downloaded binary rule, I only realized when writing my first draft of this part that the rule only targets a specific directory, not all, which is what I was supposed to do. But that didn’t matter, because shortly after constructing my rules and responses, I was faced with the reality that I can’t use my custom responses; rereading through the documentation, it says that Aurora’s Lite version only allows the usage of predefined ones. The full list of predefined responses are as follows:

Due to this restriction, I was forced to remove all custom response actions from my rules. Since my response set for the “Suspicious Binaries and Scripts in Public Folder” rule only has the custom response, it ended up going unused as a result. That leaves my “Apollo C2 Malware Executed” rule as the only custom rule left, which now looks like this:

Making Changes to the Installation
Having now established my custom rule, pushing the change is a different story. When installing Aurora Lite as a service, the service will operate based on the flags specified in the command line, as well as the files that were in the temporary folder at the time of installation. At the time I installed Aurora, I didn’t start up any custom rules and responses yet, meaning that my current installation is running without any of them active. Anytime I want to push changes to Aurora post-installation, such as making it use custom rules and activating responses, I’ll have to reinstall the service. Before doing so, first I need to add my rule to the appropriate directory in the temporary folder that I used when I first installed Aurora Lite (which in this case was C:/aurora):

Although, I’ve also added the rule under the installation:

Then, I reran the install command under the temp folder with the following flags:

After doing the re-installation, Aurora Lite is now able to execute response actions and should actively be using the “Apollo C2 Malware Executed” rule. To test the rule & response out, I went and got my Apollo binary up and running. The first time executing it, the response failed to fire off, which at the very least indicates that the detection rule is indeed working. Checking the corresponding error log, I saw the problem right away: I was using the wrong process ID field:


After re-installing the service again to echo the change, I ran the binary again for another test. The test failed again. Checking the log, this time the error was caused by the fact that the binary was run with escalated privileges (a.k.a. as root):

To fix this, I added the flag lowprivonly: false under response: of my rule. Then I ran the binary for a third time; maybe the third time’s the charm. Indeed it was, because this test succeeded:

The corresponding log in Splunk:



