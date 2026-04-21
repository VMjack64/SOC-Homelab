# Part 3: Beginning the Lab Setup (Days 3, 4)
With my servers mapped out, I began setting them up in AWS. As I don't have any previous familiarity with AWS, I spent some time learning the basics by reading the [AWS documentation](https://docs.aws.amazon.com/), as well as scouring the internet for additional resources, and modifying my diagram to reflect what I've learned.

Some important pointers: 
- In AWS, when designating a CIDR (Classless Inter-Domain Routing) block for a VPC, in addition to the first & last addresses being used by the CIDR, AWS will also need to use the second, third, and fourth IP addresses in the range (see: https://www.youtube.com/watch?v=HbTfONoekyM). 
  - This process is also applied when designating a CIDR block for a subnet within a VPC, which explains why the IP ranges in my diagram are specified as such. 
- An “Availability Zone” is just an AWS data center located in a specific region (ex: ‘us-east-2a’ is a data center located in Ohio (‘us-east-2’)).
  - Subnets within a VPC should be situated in an availability zone.
- In AWS, security groups (firewalls) are stateful, while network ACLs (NACLs) are stateless.
  - What this means is that if inbound traffic from an IP address goes through a firewall and is allowed, then responding outbound traffic directed to that IP address is also allowed, regardless of any outbound rules in place, and vice-versa.
  - By contrast, even if outbound traffic to an IP address goes through a NACL and is allowed, inbound traffic from that same IP won’t necessarily be allowed, depending on the NACL’s inbound rules. Same goes for inbound to outbound traffic.
- To establish internet connectivity for a VPC and its subnet(s) (which would convert it from a private subnet to a public subnet), the VPC and subnet(s) must allow outbound & inbound connections to/from the internet (0.0.0.0/0). Additionally, a route table must be associated with the VPC and subnet(s), which contains a route that directs outbound connections to 0.0.0.0/0 over to an internet gateway.
  - This is a perfect way to see the difference between AWS firewalls and NACLs in action.

## Home Lab Initialization
Once I learned how to work with AWS, I began the building process:
1. I started off by navigating to **Create VPC** to construct the VPC that will house my lab: 
![](/screenshots/5.png)
2. I’ve given the VPC the name `MYDFIR-30Day-SOC-Challenge`, with the following options selected for the sake of simplicity: 
![](/screenshots/6.png)
    - The **Tenancy** option can be modified to dedicate a server rack in the data center for the VPC and its EC2 instances, but I'll be leaving the selection as Default, which will borrow resources from a server currently in use to run EC2 instances in the VPC, if applicable (More about this option here: https://medium.com/@simrankumari1344/understanding-aws-tenancy-options-shared-tenancy-dedicated-hosts-and-dedicated-instances-2221bc288a9b).
    - VPC encryption wasn’t a thing at the time I created this VPC, so I’ll be ignoring it for the duration of this lab.
3. I brought internet connectivity to the VPC by creating an internet gateway in **Internet gateways**, then establishing a route table named `splunk-subnet-routetable` in **Route tables** with the following configurations: 
![](/screenshots/7.png)
![](/screenshots/8.png)
> [!NOTE]
> The `172.20.0.0/24` route above functions as a loopback to the cloud network.
4. I navigated to **Subnets** and constructed the first subnet from my diagram, `splunk-subnet`, so that the VPC can begin hosting EC2 instances:
![](/screenshots/9.png)
![](/screenshots/12.png)
![](/screenshots/13.png)
I’ve also decided on giving all subnets in the VPC internet connectivity for convenient access to the tools that'll be installed. These subnets will be associated with the same `splunk-subnet-routetable` from the previous step: 
![](/screenshots/10.png)
![](/screenshots/11.png)

## Splunk Server Creation
1. With `splunk-subnet` ready to go, I deployed the first server of this lab: The main Splunk server that will be used for log aggregation and analyzing malicious activity. On the EC2 page, I clicked on **Launch instance** to head over to the instance creation page. From there, I gave the server the following specifications:
    - Name: MYDFIR-Splunk
    - AMI: Ubuntu, Ubuntu Server 24.04 LTS
    - Architecture: 64-bit (x86)
    - Instance type: m7i-flex.large. As this server will be performing two Splunk functions, a lot of performance memory will likely be used, so higher RAM is necessary. This option is the highest I can go on AWS's free tier.
    - Key pair: RSA type, .pem format. After giving it a name, the key is downloaded onto my computer as a file; I put it in a dedicated folder for easy access.
    - Network settings:
      - VPC: MYDFIR-30Day-SOC-Challenge
      - Subnet: splunk-subnet
      - Auto-assign public IP: Enabled
      - Firewall (security groups): Create security group, with the following settings: 
        ![](/screenshots/14.png)
    - Configure storage: 1 volume only:
      - Text box: 30 GiB (highest amount of storage an instance can have on the free tier)
      - Dropdown box: Left as whatever option is selected
    - Advanced details: Ignore; all options under here aren’t necessary for this lab.
  
Under “Summary” on the right, I made sure only one instance is created, then reviewed the specs below to ensure they’re correct. After that’s done, I clicked “Launch instance” to create & start up the AWS Ubuntu virtual machine instance.
Home Lab Phase: Splunk Server Setup
With the instance up & running, connect to the instance to begin setting up Splunk on it:
Click on “Connect to instance” at the bottom of the “Successfully initiated launch of instance” screen, or click on “Instances” at the top to navigate back to the “Instances” screen, click on the instance that was just created, then click the “Connect” button.
On the “Connect” screen, click the “SSH client” tab, then copy the command under “Example:”
Open up PowerShell, then navigate to the directory where the instance’s key file is stored using ‘cd <file-path-to-directory>’.

Alternatively, open up the folder in File Explorer, then Shift + Right-click on an empty area, then click “Open PowerShell window here”.
Paste the command copied from step b, then press “Enter”. A warning prompt will appear, saying that “This key is not known by any other names” & if you want to continue connecting. Type “yes”, then press “Enter” to connect to the instance.
If connecting to the instance failed, try waiting for the initialization process to complete & make sure all status checks passed before connecting again.
Once connected, update the Ubuntu repositories by running the command: ‘sudo apt-get update && sudo apt-get upgrade -y’
Get the free trial for Splunk Enterprise from the official Splunk website (splunk.com):
On the host machine, head to Splunk’s website, then click on “Trials & Downloads”.
Under the “Splunk Enterprise” option, click “Start trial”.
If you haven’t already, create an account. Otherwise, login to your Splunk account.
On the “Choose Your Download” screen for Splunk Enterprise, click the Linux tab, then click “Copy wget link” next to the .deb package.
In the PowerShell SSH terminal, paste in the wget command, then press Enter to download the Debian package for Splunk Enterprise. This package will reside in the home (~) directory.
In the home directory (navigate via the command ‘cd ~’ if not in it already), enter the command ‘sudo dpkg -i <splunk_package_name>.deb’ to install Splunk Enterprise. By default, Splunk will be installed in the ‘/opt/splunk’ directory.
Run the command `sudo chown -R <instance username>:<instance username> /opt/splunk` to change ownership of the Splunk application to the instance’s username.
The instance’s username can be found in the SSH command from step 1b, to the left of the ‘@’: 

The reason for changing the ownership of the Splunk installation is because the owner specified isn’t the instance username upon installation (you can check with the ‘ls -l’ command; you should see 2 back-to-back columns with the name ‘splunk’ in them). Because of this, running Splunk would require root (sudo) privileges, which isn’t considered good security practice, so in order to be able to run it without root, changing the owner of the application is required. If there are any situations where ‘sudo’ was used during the installation process (asides this), it’s because it was explicitly mentioned in the official Splunk documentation to use it (https://help.splunk.com/en/splunk-enterprise/get-started/install-and-upgrade/9.4/install-splunk-enterprise-on-linux-or-macos/install-on-linux).
Navigate to the ‘/opt/splunk/bin’ directory via the command ‘cd /opt/splunk/bin’, then run `sudo ./splunk enable boot-start -user <instance username>` to enable auto starting of Splunk without root upon booting up the instance.
(If an error occurs here, check out the auto boot process of Splunk Forwarder on Linux notes)
Run the following ‘ufw’ commands:
‘sudo ufw allow 9997’
‘sudo ufw allow 8000’
These commands tell the Ubuntu instance’s internal firewalls to allow connections to ports 9997 (Splunk receiver port) & 8000 (Splunk web GUI access port).
Run Splunk using the ‘./splunk start’ command, then hold down the Space key until the agreement finishes scrolling. Alternatively, append ‘--accept-license’ at the end of the command to skip the agreement entirely. Enter in a username & password to use for the installation, then press Enter to get Splunk up & running.
Make sure to save the username & password that you entered!
On the host machine, open up a browser, and instead of using the link provided by the SSH terminal, type in the address bar ‘<splunk_instance_public_ip_address>:8000’ to connect to the Splunk web GUI remotely. Enter in the username & password from step 9 onto the login screen to access Splunk.
The link provided by the SSH terminal would be used in the instance’s browser itself. However, the Ubuntu server instance doesn’t have a browser, so connecting to it remotely is necessary, hence why the public IP should be used instead.

