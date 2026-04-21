# Part 2: Splunk vs. ELK Stack (Days 2, 6)
The first part of my diagram that I want to talk about is the Splunk setup at the bottom right:

![](/screenshots/2.png)

Since I’m using Splunk instead of the ELK stack, I'll need to determine the Splunk equivalent of each ELK component first. Conveniently, Steven discusses this in Day 2 of the series, where the mapping goes as follows:
- Elasticsearch -> Splunk indexer / search head (searching)
- Logstash -> Splunk heavy forwarder (parsing & indexing)
- Kibana -> Splunk web GUI
- Elastic agent -> Splunk universal forwarder / agent (input)

Before doing this project, I took an introductory lesson on Splunk, called Getting Data Into Splunk, where I got a good idea of the Splunk data pipeline. I corroborated the mapping above with this diagram from the lesson:

![](/screenshots/3.png)

Where the top row is equivalent to Elasticsearch, the middle row equivalent to Logstash, and the bottom row equivalent to the Elastic agents.

Next, I need to determine Splunk’s fleet server equivalent, which Steven describes in the Day 6 video as a centralized agent management system for Elastic, essentially. After some Googling, I found out that this equivalent is called a Splunk deployment server. Looking more into it in the Splunk documentation, these are some key takeaways I got:
- Deployment servers, as well as search heads & indexers, amongst others, are each installed as a separate Splunk Enterprise instance configured to perform their intended task
  - Indexer and deployment server functionality is enabled by default for newly installed Splunk Enterprise instances
  - If the total number of Splunk universal forwarders (inputs) in an environment is less than 50, an instance can be configured to perform 2 or more tasks (e.g., an instance performing searching & deployment tasks)
    - However, it’s recommended to create a dedicated Splunk instance and configure it for deployment server tasks due to the high resource usage of deployment servers
- A deployment server must be installed on a Linux machine in order to be able to manage Splunk forwarders installed on a Linux machine, in addition to Windows ones
- The Splunkd port is used for connecting to a deployment server (TCP port 8089)

Taking everything discussed into consideration, since my home lab isn’t going to be very large, I’ve decided to have two Splunk Enterprise servers in my setup, where one combines the search head & indexer functionalities, while the other is the deployment server. The Splunk setup in the first screenshot looks something like this (taken from the same lesson mentioned earlier):

![](/screenshots/4.png)

Previous Part                                | Return to Introduction                  | Next Part
-------------------------------------------- | --------------------------------------- | ---------
[Part 1: Conceptualizing the Lab](/PART1.md) | [Introduction](/README.md#introduction) | [Part 3: Beginning the Lab Setup](/PART3.md)
