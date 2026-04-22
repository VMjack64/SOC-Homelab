Previous Part                                | Return to Introduction                  | Next Part
-------------------------------------------- | --------------------------------------- | ---------
[Part 3: Beginning the Lab Setup](/PART3.md) | [Introduction](/README.md#introduction) | [Part 5: Prepare for Log Ingestion](/PART5.md)

[Part 4: Creating the Windows Server](#part-4-creating-the-windows-server-day-5)
- [Windows Server Subnet Setup](#windows-server-subnet-setup)
- [Windows Server Creation](#windows-server-creation)

# Part 4: Creating the Windows Server (Day 5)
After setting up the Splunk instance, I began building the next server on the list: A Windows server that will be exposed to the internet, essentially acting as bait & logging remote desktop login attempts from all over the world. These RDP (remote desktop protocol) brute force attack logs will be ingested into Splunk.

In Day 5 of MyDFIR’s series, Steven brings up a really good point of putting the Windows and Linux servers out of the same network as the other servers, so that should the Windows and/or Linux servers be successfully compromised by a brute force attack, the attacker(s) wouldn’t be able to discover the other servers through lateral movement. To replicate this setup in AWS, there were two options I could think of going about: Either put the Windows instance in a new VPC, or put the instance in a different subnet within the same VPC. I decided to go with the latter option, as a way to put into practice something I learned from taking the Google Cybersecurity course: Network segmentation. Simply put, this concept entails dividing the network into segments, with each part having their own security rules & access permissions. A good example is a hotel offering free Wi-Fi; the network would be divided into two parts, with one part housing the unsecure network used by its guests, while the other part houses the secure network used by the hotel staff.
## Windows Server Subnet Setup
First off, before building the Windows instance, I constructed its subnet, `unsecure-subnet`, with the following specifications:
![](/screenshots/16.png)
![](/screenshots/17.png)
![](/screenshots/18.png)

For this subnet, I’m going to create a NACL for it named `unsecure-acl`. I made sure the NACL has the following configuration before clicking create:
![](/screenshots/19.png)

After creating the NACL, I edited its inbound rules to allow all inbound connections from the internet:
![](/screenshots/20.png)

Then, I edited the NACL's outbound rules as such:
![](/screenshots/21.png)

In AWS, NACL rules are processed from lowest rule number to highest. If a matching rule is found before all rules could be processed, AWS uses that rule, and ignores the unprocessed ones. In this case, if the Windows server was compromised & it tried to make a connection to another instance on a different subnet inside the VPC (172.20.0.0/24, rule #75), that connection would be blocked, not allowed (rule #100). This setup ensures that the attacker(s) don’t discover my Splunk server & other critical infrastructure inside the VPC, while still bringing internet access to the Windows server to log any potential malicious connections.

With the NACL all set up, I modified the network ACL association for `unsecure-subnet`, associating the NACL with the subnet:
![](/screenshots/22.png)

> [!TIP]
> If the NACL doesn’t show up, click the refresh button next to the dropdown box.

## Windows Server Creation
After setting up the Windows server’s subnet, it’s time to create the instance itself. I gave this server the following specifications:
  - Name: MYDFIR-Windows-RDP
  - AMI: Windows, Microsoft Windows Server 2022 Base
  - Architecture: 64-bit (x86)
  - Instance type: t3.micro. This is the lowest option available. This server doesn’t need much processing power for its tasks.
  - Key pair: RSA type (only option), .pem format
  - Network settings:
    - VPC: MYDFIR-30Day-SOC-Challenge
    - Subnet: unsecure-subnet
    - Auto-assign public IP: Enabled
    - Firewall (security groups): Create security group, with the following settings: 
      ![](/screenshots/23.png)
  - Configure storage: 1 volume only:
    - Text box: 30 GiB
    - Dropdown box: Left as whatever option is selected

After creating the server, I left it running for 24 hours to generate RDP brute force telemetry from various IP addresses. In the meantime, I navigated to the **Security Groups** screen to create another firewall for the Windows server named `Windows-Server-Firewall` with the following specifications: 
![](/screenshots/24.png)
![](/screenshots/25.png)

Previous Part                                | Return to Introduction                  | Next Part
-------------------------------------------- | --------------------------------------- | ---------
[Part 3: Beginning the Lab Setup](/PART3.md) | [Introduction](/README.md#introduction) | [Part 5: Prepare for Log Ingestion](/PART5.md)
