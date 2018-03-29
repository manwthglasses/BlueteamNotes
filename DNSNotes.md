(Ubuntu) DNS Notes
======
### Contact
- Slack: @manwthglasses on [slack](wcscusf.slack.com)
- Email: wmurphy1@mail.usf.edu

### Table 
1. [Prerequisites](#prerequisites)
2. [Summary](#summary)
3. [Background](#background)
4. [Installation](#installation) 
5. [Configuration](#configuration)
6. [Helpful links](#helpful-links)

### Prerequisites
1. [Download virtualization program](https://www.virtualbox.org/wiki/Downloads)
2. Download ISO images
3. [Setup virtual environment](https://github.com/Nanjuan/WCSC-Blue-Team/blob/master/Network%20Adapter%20Information.md) 
4. [Setup Ubuntu ISP](https://silexone.github.io/guides/nestor/ISPsetup.html)
5. Setup PFsense box

### Summary
![box alternative text](https://github.com/manwthglasses/BlueteamNotes/blob/master/.box.png)

### Background
##### DMZ (Demilitarized Zone)
- Outward facing network inbetween trusted internal network (LAN) and untrusted external network such as the internet
- Segregated from personal files
- Typically containing devices accessible to internet traffic, such as Web and DNS servers

##### DNS (Domain Name Server)
- DNS stands for Domain Name Servers and serve the purpose of maintaining a directory of domain names and translating them to IP addresses. 
- All websites you visit are normally accessed through domain names such as "www.google.com" but are then translated to the actual IP address that will let you reach that site.
- DNS are the servers that translate the domain names to the IP addresses and even vice versa if needed. 
	- www.google.com -> 201.23.52.1
	- 201.23.52.1 -> www.google.com
- Domain Name Space 
  - The Domain Name Space is a term for the hierarchal structure in which the domain names are distributed. 
  - The Domain Name Space is an inverted tree that would read a domain name starting from right to left, and branch down by every designated dot.
    - ex: "my.usf.edu" would be found in the domain name space beginning at the root node (instead of a "/" like in UNIX filesystems, it is designated with a "."), then would branch down to the ".edu" branch, and then further to the "usf.edu" branch where the domain name would ultimately be held.
- DNS Zone: a set of DNS records for all the domain names within the specified branch, but excluding any children branches that are designated their own zones
- Configuration
	- Primary Master Server: term for reading data for a zone from a file on the server
	- Secondary Master: term for getting the zone data from another DNS server that is the Primary Master for that zone
  - Resolvers: a term for the clients that access nameservers
    - often a very simple set of libray routines that perform the basic task of querying a name server, interpreting the response from that server, and then returning the information to the program requesting it. 
    - Queries is the term for when a resolver requests an IP address or a domain name from a server. There are two different types of queries: 
      - Recursive: a recursive query is one that takes the domain requested and queries it to the root zone name server, that would then refer it to another server further down the branch, that would then keep referring it until it reached the zone for the domain. 
        ex: a recursive query for the address of "my.usf.edu" would first query the address to the root server, who would refer it to the ".edu" server, who would refer it to the ".usf" server that would be authorative for that domain and return the address.
      - Iterative" an iterative query is the component of the recursive query that gives the best answer, whether that be a referral to another server further down the branch or the actual address requested. 
- DNS Record : single entry of instructions on handling requests based on types for a zone
	- _A Record_ : Specifies IPv4 Address for a given host 
		- www.google.com -> 201.23.52.1
	- _AAAA Record_ (quad-A record): specifies IPv6 address for given host
		- www.google.com -> 2001:db8::7348
	- _CNAME Record_: specifies a domain name that has to be queried in order to resolve the original DNS query 
		- also used to create aliases of domain names
		- same server can be accessesed through documents.example.com and docs.example.com because of CNAME
	- _MX Record_: specifies a mail exchange server for a DNS domain name
		- the information is used by Simple Mail Transfer Protocol (SMTP) to route emails to proper hosts
	- _PTR Record_: (reverse of A and AAAA DNS Records) used to look up domain names based on IP addresses

- DNS Record: a single entry of instructions on handling requests for a zone (based on types)
| DNS Record Type | Description |
| --- | --- |
| _A Record_ | Specifies IPv4 Address for a given host (www.google.com translates to 201.23.51.1) |
| _AAAA Record_ (Quad-A record) | specifies IPv6 address for a given host (www.google.com translates to 2001:db8::7348) |
| _CNAME Record_ | specifies a domain name that has to be queried in order to resolve the original DNS query; used to create aliases |
| _MX Record_ | specifies a mail exchange server for a DNS domain name, used through Simple Mail Transfer protocol (SMTP) to route emails to proper hosts |
| _PTR Record_ | used to look up domain names based on IP addresses; reverse of A and AAAA records (201.23.41.1 translates to www.google.com) | 

### Installation 
- [Install Ubuntu](https://www.ubuntu.com/download/server)
- Change network options to `Host-Only, DMZ`
- Type in `ip link show` into terminal and you should see: lo, enp0s3
	- `ip link show` : shows information for all interfaces 
	- lo : loopback
	- enp0s3 : virtual network driver
- Add another interface
	- type `sudo nano /etc/network/interfaces` to edit the network interfaces configuration file with console based text editor nano
	- add 
	```
	auto enp0s3
	iface enp0s3 inet static
	address 172.20.240.23
	netmask 255.255.255.0
	gateway 172.20.240.254
	dns-nameservers 8.8.8.8
	```
	- type `sudo service networking restart`
	- verify netowrk interfaces with `ifconfig`
	- verify connectivity with `ping 8.8.8.8` 
		- `8.8.8.8` is google's DNS server


### Configuration
- [Helpful link for configuration process](https://www.ostechnix.com/install-and-configure-dns-server-ubuntu-16-04-lts/)
- [Helpful link for understanding bind](http://www.firewall.cx/linux-knowledgebase-tutorials/system-and-network-services/829-linux-bind-introduction.html)
- Update and install bind
	- `sudo apt-get update`
	- `sudo apt-get upgrade`
	- `sudo apt-get install bind9 bind9utils bind9doc dnsutils`
	- bind is a widely used domain name system software for your server to become a DNS for your network
	- dnsutils is a package for testing and troubleshooting DNS related issues, including a tool named dig 
- Configure caching name server
	- `sudo nano /etc/bind/named.conf.options`
	- uncomment and change the lines 
	```
	//forwarders {
	//	0.0.0.0;
	//};
	```
	to 
	```
	forwarders {
		8.8.8.8;
		8.8.4.4;
	};
	```
	- Because of this, if our cached server fails to recognize a IP address, our server will forward the request to 8.8.8.8 and 8.8.4.4.
		- 8.8.8.8 and 8.8.4.4 are both public Google DNS servers.  
- Change dns-nameservers so that bind will handle it 
	- `sudo nano /etc/network/interfaces`
	- change to 
	```
	auto enp0s3
	iface enp0s3 inet static
		address 172.20.240.23
		netmask 255.255.255.0
		gateway 172.20.240.254
		dns-nameservers 172.0.0.1
	```
- Refresh 
	- `sudo ip addr flush enp0s3`
		- `ip addr flush` removes all addresses for the interface `enp0s3`
	- `sudo systemctl restart networking.service`
		- `systemctl` is the system manager
		- `restart networking.service` will restart the networking service that `systemctl` manages
	- `sudo systemctl restart bind9`
		- will restart bind9
	- `ping www.google.com`
		- verify internet connection
- Creating zones
	- Adding a DNS zone, and therefore making this a Primary Master Server. 
	- `sudo nano /etc/bind/named.conf.local`
	- add
	```
	zone "wcsc.com" {
		type master;
		file "/etc/bind/forward.wcsc.com";
		allow-transfer { 172.20.241.27; };
		also-notify { 172.20.241.27; };
	};
	
	zone "20.172.in-addr.arpa" {
		type master;
		file "/etc/bind/reverse.wcsc.com";
		allow-transfer { 172.20.241.27; };
		also-notify { 172.20.241.27; };
	};
	```
	- create the zone files mentioned in the configuration
		- `sudo touch /etc/bind/forward.wcsc.com`
		- `sudo touch /etc/bind/forward.wcsc.com`
	- add zone content 
		- add lines `sudo nano /etc/bind/forward.wcsc.com`
		- ![forward.wcsc.com alternative text](https://github.com/manwthglasses/BlueteamNotes/blob/master/.forwardwcsc.jpg)
		- add lines `sudo nano /etc/bind/reverse.wcsc.com`
		- ![reverse.wcsc.com alternative text](https://github.com/manwthglasses/BlueteamNotes/blob/master/.reversewcsc.jpg)
- Verify 
	- `sudo systemctl restart bind9`
	- verify with 
	`nslookup 172.20.240.11`
		- (in this case) `nslookup` is used as a command to print the name and requested information for a domain (or host)
	`nslookup 172.20.240.23`
	`nslookup WEB.wcsc.com`
	`nslookup DNS.wcsc.com`
	`ping WEB.wcsc.com`	

### Troubleshooting
	- 'sudo named-checkconf /etc/bind/named.conf' will output the erros for fixing

### Helpful Links
	
