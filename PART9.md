# Part 9: osTicket & Splunk (Days 23, 24, 25)
Wanting more of the SOC analyst experience, I’ve decided that I was going to try and integrate a ticketing system with Splunk. I will be using osTicket for this, since MyDFIR also used it in his series (and because it's free). For the setup process, I largely followed the Day 24 video, starting off with creating the instance where osTicket will be installed in, giving it the following specifications:
  - Name: MYDFIR-osTicket
  - AMI: Windows, Microsoft Windows Server 2022 Base
  - Architecture: 64-bit (x86)
  - Instance type: t3.micro
  - Key pair: RSA type, .pem format
  - Network settings:
    - VPC: MYDFIR-30Day-SOC-Challenge
    - Subnet: splunk-subnet
    - Auto-assign public IP: Enabled
    - Firewall (security groups): Create security group, with the following settings:
      ![](/screenshots/163.png)
      ![](/screenshots/164.png)
  - Configure storage: 1 volume only:
    - Text box: 30 GiB
    - Dropdown box: Left as whatever option is selected

After creating the instance, I modified the `30-Day-MYDFIR-SOC-Challenge` firewall to add an inbound rule allowing all traffic from the osTicket instance:
![](/screenshots/165.png)

Once connected to the instance via remote desktop, I continued following along with the video, installing XAMPP onto the instance. This is where I made a few changes of my own: For all instances where I’m supposed to use the public IP address of the osTicket instance, I instead used the localhost address (127.0.0.1). Since the public IP address of the instance changes everytime I turn it off and then turn it back on, it wouldn't be feasible for me to use the public IP, as that would necessitate navigating through a lot of settings to make changes per every shut down I do. Because of this difference, I didn’t make any adjustments to the XAMPP properties configuration file, and I didn’t have to do the IP address authorization step (although I did provide a password for the root account).

With this in mind, I got XAMPP all set up and followed the rest of the video to install osTicket. With osTicket installed, I get two URLs:
  - The osTicket URL (http://127.0.0.1/osticket/upload/)
  - The staff control panel URL (http://127.0.0.1/osticket/upload/scp)

If I want to access osTicket from my host computer, I must replace the localhost address in these URLs with the public IP address that's currently listed in AWS for the osTicket instance. Doing so for the staff control panel URL and pasting it into my host computer’s browser, I was able to access osTicket after logging in with the credentials I provided in the **Admin User** section during the osTicket installation process.

## Setting up osTicket Auto Startup
Now that I can access osTicket, I did a quick test to see if XAMPP will automatically start up by default. It doesn’t, but I wanted to set up XAMPP to make that happen so it’d remove the hassle of having to manually turn on the Apache and MySQL servers via the XAMPP Control Panel every time I boot up the osTicket instance in order to access osTicket. I looked up the internet for ways I could achieve this, and I found this YouTube video to be of help:
Opening up the XAMPP Control Panel, I clicked the “Config” button:

Under the “Autostart of modules” section, I checked “Apache”, “MySQL”, & “Start Control Panel Minimized”:

Click Save, then close the XAMPP Control Panel
Navigate to the xampp directory (typically installed under C:\)
Under the xampp directory, I located the xampp-control application, then right-clicked on it and selected “Create shortcut”. Then I temporarily moved the new shortcut icon to the desktop:

Open the “Run” application by right-clicking on the Windows button on the taskbar, then clicking “Run”
With the “Run” application up, I typed in shell:startup in the text box, then clicked OK. This opens the Startup folder 
I moved the shortcut created in step 5 to the Startup folder:

Repeat step 5 to create another xampp-control shortcut that will be moved into the desktop
Right-clicking on the shortcut created in the previous step, I clicked on Properties to open the Properties menu
In the Properties menu, open the dropdown box next to “Run:”, then click “Minimized”
Click OK to apply the changes and close the Properties menu
Restart the instance
Upon logging into the instance, I see the xampp-control app icon pop up in the “^” menu automatically, indicating the process was successful:

With XAMPP now able to start on its own, I wanted to see if I can access osTicket after booting up the instance without having to log into it. So I turned off the instance, then turned it back on. After waiting a few minutes, I then tried to access osTicket, but was unable to, meaning I do have to log into the instance first in order to trigger the auto start for XAMPP. Nonetheless, this should at least minimize some of the tedious work in the future.

## A Blind Shot at Integrating osTicket with Splunk
With XAMPP and osTicket now properly set up, it’s time to try integrating osTicket with Splunk. Booting up & connecting to the main Splunk instance, my idea is to modify my existing Splunk alerts to create tickets based on what event went off. While working with Splunk’s alert system to create the alerts for authentication activity and malware, I saw there was a built-in webhook alert action, and initially had the idea of using it to pull this off, but when it came time to use it, I find that it’s very limited in terms of functionality; it only allows me to specify the site to make the POST request to. According to the osTicket API documentation, an osTicket API key is required in order to make the integration work, so the native webhook action isn’t going to be of use here.
With my one viable option rendered inviable, I turned to the internet to try and find any more leads on completing this task. Along the way, I found this pretty good YouTube tutorial on integrating a 3rd party ticketing system with Splunk. According to the video, the general idea on how to perform the integration is through a Splunk extension based on the 3rd party ticketing system used and making sure the ticketing system allows API access, which osTicket does. So, after following the first 2 minutes of the Day 25 video to generate my osTicket API key, I went into Splunk and searched the Splunk add-on store for osTicket-Splunk add-ons and found a couple of interest: “osTicket Addon - Support Ticket System” and “HTTP Alert Action”.
Add-on #1: “osTicket Addon - Support Ticket System” (failed)
DISCLAIMER: I omitted much of my experiences with the “osTicket Addon - Support Ticket System” add-on from my notes, asides from the fact that it didn’t work for me, so take this sub-section with a grain of salt, as I might mis-remember some things here.
The osTicket add-on has limited documentation in the “Details” section of its Splunkbase page, yet it seemed explanatory enough after reading through it, so I installed it, hopeful that it would accomplish my goal. Unfortunately, this didn’t happen; the only documentation I could find for the add-on was in the “Details” section on its Splunkbase page. While much of it was fairly clear, the configuration step-by-step process was where things went downhill:

I used the Day 25 video as a blueprint here, entering the following into the following fields:
Api Endpoint: http://172.20.0.26/osticket/upload/api/tickets.xml (172.20.0.26 is the private IP address of the osTicket instance)
Host1: 172.20.0.30 (private IP address of MYDFIR-Splunk, which houses the Splunk search head component)
Api1: The API key that shows up in osTicket from following the first 2 minutes of the Day 25 video
Using the private IP addresses of the osTicket & MYDFIR-Splunk instances here is viable since both instances are situated in the same subnet & VPC. After saving these changes, I navigated to my alerts and modified my authentication alert to use the osTicket add-on as an alert action. Then after triggering the alert, I checked the add-on’s dashboard for any changes, but it remained the same. I also checked osTicket to see if a new ticket popped up, but there wasn’t. Looking at the add-on documentation again, I saw that step 1 mentions specifying the location of the osTicket installation after the URL, so I tried following it as described and modified the “Api Endpoint” field, removing the “/upload/api/tickets.xml” part to no avail, then the “osticket/upload/api/tickets.xml” part to no avail either. I even made sure ports 80 and 443 were allowed on both instances involved, but nothing changed after rerunning the setups mentioned here.
Add-on #2: HTTP Alert Action (success)
With the previous add-on failing on me, I dumped it for the next one, which I came across by coincidence while searching for related add-ons. Like the previous add-on, this also has a similar level of documentation, viewable only on the add-on’s Splunkbase page under the  “Details” section. Looking at the sample screenshot, documentation, and the add-on name, I found this add-on very promising in accomplishing this task. After installing the add-on, I navigated to my alerts and modified my authentication alert (by clicking “Edit” under the alert, then “Edit alert”) to use the add-on as an alert action (specified as “Send HTTP request”):

Then, under the “Send HTTP request” action, I inputted the following details:
Endpoint: http://172.20.0.26/osticket/upload/api/tickets.xml
Headers: X-API-Key=<API key mentioned in the Api1 field from the previous add-on>
Headers Key-Value Separator: Left blank. Depending on the API key, a value may need to be inputted here.
Body: The XML example under “XML Payload Example” from the osTicket GitHub
The text box formats the payload into a single line, so I first pasted it into an empty Notepad document, made any adjustments through there, then copy & pasted the edited payload into the text box.
Method: POST
Verify SSL Certificate: False. osTicket API verification is done through the headers specified above & the HTTP protocol.

After saving the changes, I triggered the alert, then checked osTicket, feeling confident that there’s a new ticket since I was able to imitate the webhook setup in the Day 25 video; no new ticket showed up, however. Checking the IP addresses of the instances, I didn’t notice any abnormalities. Dumbfounded, I finally tried searching the Splunk internal logs to debug the issue (which looking back, I should’ve done the same when troubleshooting the first add-on). I found the osTicket connection log associated with the add-on, and thankfully the error was nothing too complicated - just a simple URL conversion from http to https:

After making the changes to the endpoint, I retriggered the alert and checked osTicket to see if a new ticket popped up… and it did:


After making the same changes with the malware alert, the osTicket-Splunk integration is now a success. Though, one question loomed in my mind: With how the integration is set up, I thought of the following scenario: Say I have a lot of alerts with no action to create a ticket in osTicket upon triggering. One day, I decided that I’d want to edit ALL these alerts to push to osTicket upon triggering. This would’ve been very tedious and time consuming, which became the basis for an idea I initially had, which was to implement a bulk alert editing system that would streamline the process of making any adjustments like this. I tried searching the Splunk add-on store for any add-ons that would enable me to do this, but wasn’t able to find any. I tried searching the internet for any solutions, but couldn’t find any that are applicable with my skill level. While continuing to struggle with finding a solution, I had a thought: All of these alerts have unique configurations, each tailored for a specific kind of machine, like a Windows machine or a Linux server. If I were to bulk edit these alerts to add the alert action, then theoretically speaking, wouldn’t that also mess with their configurations in the process, resulting in me having to edit each alert individually to fix them? To me, this alone was a very compelling argument to get me to consider rescinding this plan, but while looking online, I found this post on Reddit that seemed to be another convincing argument against implementing a mass alert modification system. Specifically, a few replies mentioned how having around 600 alerts running at once will overload the Splunk search head, causing Splunk to lag as a result; having a small amount of optimized alerts would be much more manageable for the software. Ultimately, these two arguments did convince me to drop the mass alert editing plans and just move on.
