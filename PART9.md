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
Now that osTicket is accessible, I wanted to set up XAMPP to automatically start upon booting up the instance. By default, it doesn't, but doing this would eliminate the hassle of navigating to the XAMPP Control Panel and manually turning on the Apache and MySQL servers whenever I want to access osTicket. Searching the internet for solutions, I found [this YouTube video](https://www.youtube.com/watch?v=-y2YWq7wvPA:) to be of help:
1. Opening up the XAMPP Control Panel, I clicked Config, then under the **Autostart of modules** section, I checked “Apache”, “MySQL”, & “Start Control Panel Minimized”:

![](/screenshots/166.png)

2. After saving, I closed the XAMPP Control Panel, then navigated to the `xampp` directory (typically under `C:\`).
3. Under the `xampp` directory, I located the `xampp-control` application, right-clicked on it, and selected “Create shortcut”. I temporarily moved the new shortcut to the desktop:
![](/screenshots/167.png)
4. Right-clicking the Windows button on the taskbar, I selected “Run” to open the Run application.
5. With the Run application up, I typed `shell:startup` in the text box, then clicked OK to open the Startup folder. I moved the shortcut created in step 3 to the Startup folder.
6. I repeated step 3 to create another `xampp-control` shortcut, which will be moved to the desktop.
7. Right-clicking on the shortcut created in the previous step, I clicked on Properties to open the Properties menu.
8. In the Properties menu, I opened the dropdown box next to “Run:”, then selected “Minimized”.
9. I clicked OK to apply the changes and close the Properties menu. Afterwards, I restarted the instance.

Upon logging into the instance, I saw the `xampp-control` app icon pop up automatically, indicating the process was successful:
![](/screenshots/168.png)

But, when rebooting the instance and testing to see if I could access osTicket without having to log into the instance, the test failed, meaning I do need to log into the instance at the very least if I want the auto start to trigger. Nonetheless, this should minimize some of the hassle, so I'm still satisfied with how this turned out.

## A Blind Shot at Integrating osTicket with Splunk
With XAMPP & osTicket now ready, I began my attempts to integrate the software with Splunk. In MYDFIR-Splunk's web GUI, my idea here is to modify my existing Splunk alerts to create tickets whenever the alert fires. Initially, I had the idea of using the built-in webhook alert action, but its highly limited functionality ended up making it incompatible with osTicket. According to [osTicket's API documentation](https://docs.osticket.com/en/latest/Developer%20Documentation/API%20Docs.html#resources), an API key must be specified in order to make the integration work, but the native webhook action only allows me to specify the site to make the POST request to.

As the webhook option became inviable to me, I turned to the internet for more leads. That's when I came across [this YouTube tutorial](https://www.youtube.com/watch?v=JD14OCh8mg4), in which the general idea on performing the integration is through a Splunk add-on based on the ticketing system used and hoping the system allows API access, which osTicket thankfully does. After generating my osTicket API key, I went into Splunk and searched the add-on store for osTicket add-ons. I found a couple for my situation: **osTicket Addon - Support Ticket System** and **HTTP Alert Action**.

### Add-on #1: “osTicket Addon - Support Ticket System” (failed)
When trying this add-on, I ran into a roadblock with the configuration step-by-step process:
![](/screenshots/169.png)

This is what I inputted into the following fields:
  - `Api Endpoint`: http://172.20.0.26/osticket/upload/api/tickets.xml (where 172.20.0.26 is the private IP address of my osTicket instance)
  - `Host1`: 172.20.0.30 (private IP address of MYDFIR-Splunk, which houses the search head component)
  - `Api1`: The API key I just generated in osTicket

Since the osTicket & MYDFIR-Splunk instances are situated in the same subnet & VPC, using their private IP addresses is viable for me here.

After saving these changes, I navigated to my alerts and modified my authentication alert to use the osTicket add-on as an alert action. Then after triggering the alert, I checked the add-on’s dashboard for any changes, but it remained the same. I also checked osTicket to see if a new ticket popped up, but there wasn’t. Looking at the add-on documentation again, I saw that step 1 mentions specifying the location of the osTicket installation after the URL, so I tried following it as described and modified the “Api Endpoint” field, removing the “/upload/api/tickets.xml” part to no avail, then the “osticket/upload/api/tickets.xml” part to no avail either. I even made sure ports 80 and 443 were allowed on both instances involved, but nothing changed after rerunning the setups mentioned here.

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
