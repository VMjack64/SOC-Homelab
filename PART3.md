Previous Part                                | Return to Introduction                  | Next Part
-----------------------------------------    | --------------------------------------- | ---------
[Part 2: Splunk vs. ELK Stack](/PART2.md)    | [Introduction](/README.md#introduction) | [Part 4: Creating the Windows Server](/PART4.md)

[Part 3](#part-3-beginning-the-lab-setup-days-3-4)
- [Home Lab Initialization](#home-lab-initialization)
- [Splunk Server Creation](#splunk-server-creation)
- [Splunk Server Setup](#splunk-server-setup)

# Part 3: Beginning the Lab Setup (Days 3, 4)
With my servers mapped out, I began setting them up in AWS. As I don't have any previous familiarity with AWS, I spent some time learning the basics by reading the [AWS documentation](https://docs.aws.amazon.com/), as well as scouring the internet for additional resources, and modifying my diagram to reflect what I've learned.

Some important pointers: 
- In AWS, when designating a CIDR (Classless Inter-Domain Routing) block for a VPC (Virtual Private Cloud), in addition to the first & last addresses being used by the CIDR, AWS will also need to use the second, third, and fourth IP addresses in the range (see: https://www.youtube.com/watch?v=HbTfONoekyM). 
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
With `splunk-subnet` ready to go, I deployed the first server of this lab: The main Splunk server that will be used for log aggregation and analyzing malicious activity. On the EC2 page, I clicked on **Launch instance** to head over to the instance creation page. From there, I gave the server the following specifications:
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
  
After reviewing the specifications under **Summary** on the right and ensuring they’re correct, I clicked **Launch instance** to create & start up the AWS virtual machine instance.
## Splunk Server Setup
Once the instance is up & running, I connected to it by navigating to the **Connect** screen for the instance, then following the prompts on-screen for connecting via SSH. After connecting to the instance, I updated the Ubuntu repositories first by running the command `sudo apt-get update && sudo apt-get upgrade -y`.
1. After updating the repositories, I began the process of setting up Splunk on the instance. I opened up the browser on my laptop's taskbar (a.k.a. on my host machine) and went to the [official Splunk website](https://www.splunk.com) to get the free trial for Splunk Enterprise:
    - On the "Choose Your Download" screen, on the "Linux" tab, click "Copy wget link" next to the `.deb` package. This command will be used in the PowerShell SSH terminal.
2. In the PowerShell terminal for the current SSH session, I pasted & entered in the wget command from the previous step to download the Debian package for Splunk Enterprise, which will reside in the home (~) directory.
3. After navigating to the home directory (via the command `cd ~` if not in it already), I entered the command `sudo dpkg -i <splunk_package_name>.deb` to install Splunk Enterprise onto the Ubuntu instance. By default, Splunk is installed into the `/opt/splunk` directory.
4. I ran the command `sudo chown -R <instance username>:<instance username> /opt/splunk` to change ownership of the Splunk installation to the instance’s username, to which the latter can be found in the SSH command used during the connection process, left of the ‘@’: 
![](/screenshots/15.png)
> [!NOTE]
> The reason for changing the ownership is in regards to running the Splunk software. The owner specified for the software isn’t the instance username upon installation (checkable via the `ls -l` command). Because of this, I won't be able to run Splunk unless I utilize root user (`sudo`) privileges. However, running software as root isn’t considered good security practice, hence the change.
5. After navigating to the `/opt/splunk/bin` directory, I ran `sudo ./splunk enable boot-start -user <instance username>` to allow Splunk to auto start without root upon booting up the instance.
> [!NOTE]
> If a problem arises here, check out the [Installing Linux UF section in Part 10](/PART10.md#installing-linux-uf).
6. I ran the following `ufw allow` commands to tell the Ubuntu instance's internal firewall to allow connections to/from the following ports:
    - `sudo ufw allow 9997` (Splunk receiver port)
    - `sudo ufw allow 8000` (Splunk web GUI access port)
7. I finally ran Splunk by entering the command `./splunk start`. A terms of agreement pops up; I held down the space bar key until I reached the end. After entering in a username & password to use for the installation, I pressed Enter to get Splunk up & running.
> [!NOTE]
> Alternatively, I could've appended ` --accept-license` at the end of the `./splunk start` command to skip the agreement automatically.

> [!IMPORTANT]
> Make sure to make a note of the username & password entered!

8. On my host machine, I opened up my browser, and instead of using the link provided in the SSH terminal, I typed into the address bar `<splunk-instance_public-ip-address>:8000` to reach Splunk's web GUI remotely. On the login screen, I entered in the username & password from step 7 to access the main Splunk interface.

Previous Part                                | Return to Introduction                  | Next Part
-----------------------------------------    | --------------------------------------- | ---------
[Part 2: Splunk vs. ELK Stack](/PART2.md)    | [Introduction](/README.md#introduction) | [Part 4: Creating the Windows Server](/PART4.md)
