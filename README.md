# DMZ Exploitation
This week, you will apply the techniques you learned in class to pivot across subnets and attack a DMZ.

## Background on DMZs
Most commonly, machines on a LAN send requests to the public Internet through a router, which forwards requests to the web after performing NAT, and forwards responses back to the requesting host on the LAN. This router typically has a firewall that only allows connections to/from certain ports, such as 80 and 443 (HTTP and HTTPS) .

Network engineers can add an additional layer of protection to the network by installing a **DMZ (demilitarized zone)**, which acts as an additional layer of protection in front of an existing network.

Only a _single_ machine separates the LAN pictured above from the public Internet: the `pfSense` router. In a DMZ, a whole _network sits in between.

Recall that when machines on a LAN communicate on the Internet, they _all_ forward packets straight through the **default gateway**. When data comes back from the Internet, the gateway filters and forward it to the proper host on the LAN.

Data passes through two gateways on its way through a DMZ. First, it passes through a router from the Internet into a network called the DMZ.  This router typically runs a firewall that filters out all traffic that the organization doesn't expect—e.g., it may only allow connections to HTTP/HTTPS, email, and SSH ports into the DMZ.

  ![Dual-firewall DMZ](https://upload.wikimedia.org/wikipedia/commons/thumb/6/60/DMZ_network_diagram_2_firewall.svg/960px-DMZ_network_diagram_2_firewall.svg.png)
  **Note**: This is Creative Commons: https://commons.wikimedia.org/wiki/File:DMZ_network_diagram_2_firewall.svg

From there, machines on the DMZ can forward requests to machines on the  internal network. Sensitive data is typically stored only in the internal network, _not_ on machines in the DMZ. Clients on the public Internet interact with that data by sending requests to the machines Interaction with that data is done _through_ machines in the DMZ, which act as proxies. This means that attackers who compromise a machine in the DMZ do _not_ immediately own the machines in the internal network, and because it gives network administrators fine control over _exactly_ how services from the internal network are exposed to the outside world.

One well-motivated use case for a DMZ is that of public web servers in a DMZ communicating with databases in an internal network.

---

In the topology above, imagine the machines in the DMZ are HTTP servers, and those on the intranet are databases. These web servers listen for requests from the public Internet, then query databases themselves in order to fulfill HTTP requests. Note that this **dual-firewall architecture** ensures that machines on the Internet can never interact directly with hosts on the internal network—everything must get filtered through the DMZ.

This gives administrators granular control over which machines on the intranet can connect to which machines in the DMZ. In other words, administrators can configure firewall policies that allow machines in the DMZ to talk _only_ to machines in the intranet they're known to depend on. This way, compromising a machine in the DMZ does _not_ grant the attacker  access to all of the machines on the intranet.

While DMZs add an extra layer of protection to a network, they are not impenetrable. Attackers can still exploit machines on the intranet, Attackers can still exploit machines on the internal network by compromising a machine in the DMZ and using _it_ to send traffic to the intranet. In other words, attackers can still exploit machines on the internal network by compromising a machine in the DMZ, and using it as a _router_ to the internal network—they send malicious traffic to their victim machine in the DMZ, and forward it from the victim to the intranet, bypassing the DMZ firewalls. This technique is called **pivoting**, because attackers must route data _through_ a _pivot point_ in order to communicate with a target network.

Suppose an attacker tries to penetrate the databases in the topology we've been discussing. We have the following scenario:
- The attacking machine is on the public Internet at `72.54.18.19`.
- The router in front of the target network is at`64,54,74.83`.
- The servers in the DMZ have IP addresses in the range `192.168.16.0/24`.
- The databases in the intranet have IP addresses in the range `172.16.0.0/24`.
- All of the servers in the DMZ can reach all of the databases in the intranet.

Suppose a hacker compromises a machine in the DMZ. Their attacking machine can't hit the intranet databases, _but the compromised host in the DMZ can_. This means they can open a shell on their victim in the DMZ, and use the _victim_ to launch their attacks. For example, an attacker might gain access to a server in the DMZ; upload `nmap`; and port-scan the subnet from the compromised host _instead_ of from the attacking machine.

While such manual methods are effective, Metasploit provides a module that automate performing scans and sending exploits through pivot points: `post/multi/manage/autoroute`.

#### Understanding Pivots
Recall that a Meterpreter session is an open connection to a compromised machine. Sometimes, compromised machines have interfaces to networks the attacking machine cannot access. Consider the example above—machines in the DMZ had interfaces to the `172.16.0.0/24` internal subnet. The attacking machine did _not_ have an interface to this network.

However, if the attacker gets a shell on a machine in the DMZ, they can send messages from the victim directly to machines on the intranet. But _they will still not be able to communicate with the intranet directly from their attacking machine_, because it doesn't have an interface to the intranet. This means they have to log into a shell on the victim to interact with the internal network. But this constrains them to using only the tools installed on that host—which is obviously a problem, because all of their tools are on their attacking machine.

Metasploit addresses this issue by allowing you to use a compromised host as a _router_. This enables you to send traffic _through_ a Meterpreter session to a victim, and _into_ the foreign subnet.

In our case, the machines on the network have IP addresses in the range `172.16.0.0/24`. Suppose an attacker compromises `172.16.0.4`. This machine has an interface to the internal `192.168.0.0/24` network. The attacker can configure Metasploit to use `172.16.0.4` as a _proxy server_, and route any traffic to IP address in `192.168.0.0/24` _through_ `172.16.0.4`.

This means that Metasploit will send any packets destined for `192.168.0.0/24` addresses to `172.16.0.4` _first_. There, Meterpreter will perform NAT to make it look like traffic is originating from the compromised host. Responses are forwarded through `172.16.0.4` back to the attacking machine.

In other words: After you set up a route to `192.168.0.0/24` through `172.16.0.4`, Metasploit will automatically launch scans against IP addresses in the range `192.168.0.0/24` _from the machine at `127.16.0.4`_. This means that you can launch a port scan against `192.168.0.0/24` from your _attacking_—machine, and Metasploit will automatically route packets through the victim.

Bouncing scans and exploits through pivot points sounds complicated, and it is. Fortunately, Metasploit has a built-in module that makes this relatively easy: `post/multi/manage/autoroute`.

To add a route through `172.16.0.4` to `192.168.0.0/24`, you simply run the commands below. Note that you only need to set the `SESSION` to run the module on. This tells Metasploit which connection to use to tunnel data to the new subnet.

You can configure the IP addresses and netmask of the target subnet manually, but Metasploit will typically be able to add routes automatically.

  ```bash
  msf > use post/multi/manage/autoroute
  msf post(multi/manage/autoroute) > set SESSION 1
  msf post(multi/manage/autoroute) > run
  ```

You'll learn more about the `post/multi/manage/autoroute` module and pivoting in the homework assignment and during next week's session.

---
## Instructions
In this assignment, you'll use everything you learned in class to enumerate and exploit hosts in a **review** scenario very similar to that from class. Then, you'll **expand** on what you learned by using Metasploit's built-in `post/multi/manage/autoroute` module to scan across subnets.

Launch the [DMZ Exploitation Capstone Lab](https://cybrscore.learnondemand.net/Lab/23930), and work from **Part 1 — Scan Known Web-Facing Target IP** through **Part 4 — Covering Tracks** using the Kali **WAN** VM. The **Bonus Tasks** are not required, but are _highly_ encouraged.

Don't hesitate to reach out to your instructional staff and classmates for help.

**Good luck!**
