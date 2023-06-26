# Project-Base-Learning-8
LOAD BALANCER SOLUTION WITH APACHE

## PROJECT TASK

- Deploy and configure an Apache Load Balancer for **Tooling Website** solution on a separate Ubuntu EC2 instance. Make sure that users can be served by Web servers through the Load Balancer.

## BACKGROUND KNOWLEDGE

### LOAD BALANCER SOLUTION WITH APACHE

- Aside Apache, nginx can also be used for load balancer
- 
- After completing [Project-7](https://gitlab.com/users/sign_in) you might wonder how a user will be accessing each of the webservers using 3 different IP addreses or 3 different DNS names. You might also wonder what is the point of having 3 different servers doing exactly the same thing.

- When we access a website in the Internet we use a [URL](https://en.wikipedia.org/wiki/URL) and we do not really know how many servers are out there serving our requests, this complexity is hidden from a regular user, but in case of websites that are being visited by millions of users per day (like Google or Reddit) it is impossible to serve all the users from a single Web Server (it is also applicable to databases, but for now we will not focus on distributed DBs).

- Each URL contains a [domain name](https://en.wikipedia.org/wiki/Domain_name) part, which is translated (resolved) to IP address of a target server that will serve requests when open a website in the Internet. Translation (resolution) of domain names is perormed by [DNS servers](https://en.wikipedia.org/wiki/Domain_Name_System), the most commonly used one has a public IP address 8.8.8.8 and belongs to Google. You can try to query it with [nslookup](https://en.wikipedia.org/wiki/Nslookup) command:

Run below command on your running DB Server from Project 7

`nslookup 8.8.8.8`

![8_1](https://github.com/EzeOnoky/Project-Base-Learning-8/assets/122687798/6dd4ea03-b15a-4d18-bb27-ab33978627ef)

- When you have just one Web server and load increases – you want to serve more and more customers, you can add more CPU and RAM or completely replace the server with a more powerful one – this is called **"vertical scaling"**. This approach has limitations – at some point you reach the maximum capacity of CPU and RAM that can be installed into your server.

- Another approach used to cater for increased traffic is **"horizontal scaling"** – distributing load across multiple Web servers. This approach is much more common and can be applied almost seamlessly and almost infinitely (you can imagine how many server Google has to serve billions of search requests).

- Horizontal scaling allows to adapt to current load by adding (scale out) or removing (scale in) Web servers. Adjustment of number of servers can be done manually or automatically (for example, based on some monitored metrics like CPU and Memory load).

- Property of a system (in our case it is Web tier) to be able to handle growing load by adding resources, is called ["Scalability"](https://en.wikipedia.org/wiki/Scalability).

- In our set up in Project-7 we had 3 Web Servers and each of them had its own public IP address and public DNS name. A client has to access them by using different URLs, which is not a nice user experience to remember addresses/names of even 3 server, let alone millions of [Google servers](https://en.wikipedia.org/wiki/Google_data_centers).

- In order to hide all this complexity and to have a single point of access with a single public IP address/name, a [Load Balancer](https://en.wikipedia.org/wiki/Load_balancing_(computing)) can be used. A Load Balancer (LB) distributes clients’ requests among underlying Web Servers and makes sure that the load is distributed in an optimal way.

### Side Self Study

- Read about different [Load Balancing concepts](https://www.nginx.com/resources/glossary/load-balancing/) and difference between [L4 Network LB](https://www.nginx.com/resources/glossary/layer-4-load-balancing/) and [L7 Application LB](https://www.nginx.com/resources/glossary/layer-7-load-balancing/).

- Let us take a look at the updated solution architecture with an LB added on top of Web Servers (for simplicity let us assume it is a software L7 Application LB, for example – [Apache](https://httpd.apache.org/docs/2.4/mod/mod_proxy_balancer.html), [NGINX](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/) or [HAProxy](https://http://www.haproxy.org/)).

![8_2](https://github.com/EzeOnoky/Project-Base-Learning-8/assets/122687798/23941292-91ce-49cc-b94e-a5060ab97e3a)

## Setup and technologies used in Project 8
- In this project we will enhance our **Tooling Website** solution by adding a Load Balancer to disctribute traffic between Web Servers and allow users to access our website using a single URL.

- To simplify, let us implement this solution with **2 Web Servers**, the approach will be the same for 3 and more Web Servers.
- Make sure that you have following servers installed and configured within Project-7:

1. - Two RHEL8 Web Servers

2. - One MySQL DB Server (based on Ubuntu 20.04)

3. - One RHEL8 NFS server

![8_3](https://github.com/EzeOnoky/Project-Base-Learning-8/assets/122687798/4492fd2b-dd0c-4f7d-a978-dc63fdad961e)

## ***CONFIGURE APACHE AS A LOAD BALANCER***

## 1 
- I Created an Ubuntu Server 20.04 EC2 instance and named it Project-8-apache-lb, below is the look of my EC2 list:

<img width="736" alt="ECs running" src="https://user-images.githubusercontent.com/115954100/225525386-220a40b6-23ac-4e52-9ca4-997876b31b0d.png">

## 2 
- I opened TCP port 80 on Project-8-apache-lb by creating an Inbound Rule in Security Group.

## 3
- I installed Apache Load Balancer on Project-8-apache-lb server and configure it to point traffic coming to LB to both Web Servers:

```
#Install apache2 and its dependencies
sudo apt update
sudo apt install apache2 -y
sudo apt-get install libxml2-dev

#Enable following modules that will enable the apache work well:
sudo a2enmod rewrite
sudo a2enmod proxy
sudo a2enmod proxy_balancer
sudo a2enmod proxy_http
sudo a2enmod headers
sudo a2enmod lbmethod_bytraffic

#Restart apache2 service
sudo systemctl restart apache2
```

- Make sure apache2 is up and running.

`sudo systemctl status apache2`

- **Configure load balancing**


#On below path is the default config file of Apache2, Enter this config file and make some changes

`sudo vi /etc/apache2/sites-available/000-default.conf`

#Add this configuration into this section <VirtualHost *:80>  </VirtualHost>...

#This added file is telling the apache server to map the web servers IP to the loadbalancer such that we can reach the web servers from the LB & the LB can efficiently distribute traffic to the Web Servers

```
#Scripted
<Proxy "balancer://mycluster">
               BalancerMember http://<WebServer1-Private-IP-Address>:80 loadfactor=5 timeout=1
               BalancerMember http://<WebServer2-Private-IP-Address>:80 loadfactor=5 timeout=1
               ProxySet lbmethod=bytraffic
               # ProxySet lbmethod=byrequests
        </Proxy>

        ProxyPreserveHost On
        ProxyPass / balancer://mycluster/
        ProxyPassReverse / balancer://mycluster/
        
#Executed
<Proxy "balancer://mycluster">
               BalancerMember http://172.31.88.49:80 loadfactor=5 timeout=1
               BalancerMember http://172.31.89.61:80 loadfactor=5 timeout=1
               ProxySet lbmethod=bytraffic
               # ProxySet lbmethod=byrequests
        </Proxy>

        ProxyPreserveHost On
        ProxyPass / balancer://mycluster/
        ProxyPassReverse / balancer://mycluster/
```

![8_11](https://github.com/EzeOnoky/Project-Base-Learning-8/assets/122687798/a3500857-2479-4f24-9469-9a11a1449857)

Above is telling the Loader Balance to mapped the Web server private IP such that we can reach the web server from the loader balancer, the loader balancer can also server to distribute traffic to the web server efficiently

#Restart apache server

`sudo systemctl restart apache2`


![8_4](https://github.com/EzeOnoky/Project-Base-Learning-8/assets/122687798/de0698f9-4d78-4052-b102-b194e5432a68)

In Apache, there are different type of Load Balancing Methods - for our project, we are using load balancing by traffic
- [bytraffic](https://httpd.apache.org/docs/2.4/mod/mod_lbmethod_bytraffic.html) balancing method will distribute incoming load between your Web Servers according to current traffic load. We can control in which proportion the traffic must be distributed by **loadfactor** parameter.

- You can also study and try other methods, like: [bybusyness](https://httpd.apache.org/docs/2.4/mod/mod_lbmethod_bybusyness.html), [byrequests](https://httpd.apache.org/docs/2.4/mod/mod_lbmethod_byrequests.html), [heartbeat](https://httpd.apache.org/docs/2.4/mod/mod_lbmethod_heartbeat.html).

## 4
- I verified that my configuration works – by accessing my LB’s public IP address or Public DNS name from my browser:

```
Script
http://<Load-Balancer-Public-IP-Address-or-Public-DNS-Name>/index.php

Executed
http://<Load-Balancer-Public-IP-Address-or-Public-DNS-Name>/index.php
```

- **Note:** If in the Project-7 you mounted /var/log/httpd/ from your Web Servers to the NFS server – unmount them and make sure that each Web Server has its own log directory.

Use below command to achieve the unmounting

`sudo umount -f /var/log/httpd`

- I opened two ssh/Putty consoles for my running Web Servers and ran following command to access the log file, the log will show events, people wo tried to access the web server via the LB:

`sudo tail -f /var/log/httpd/access_log`

- Try to refresh your browser page from different terminals and check the access log file also, i saw the terminal that tried to access the web page, the location of the terminal etc

![8_5](https://github.com/EzeOnoky/Project-Base-Learning-8/assets/122687798/78cc0046-6ed8-4108-b522-dd3141882c63)

- http://<Load-Balancer-Public-IP-Address-or-Public-DNS-Name>/index.php several times and make sure that both servers receive HTTP GET requests from your LB – new records must appear in each server’s log file. The number of requests to each server will be approximately the same since we set **loadfactor** to the same value for both servers – it means that traffic will be disctributed evenly between them.
  
  **Side Self Study:**
  
- Read more about different configuration aspects of [Apache mod_proxy_balancer module](https://httpd.apache.org/docs/2.4/mod/mod_proxy_balancer.html). Understand what sticky session means and when it is used.
  
**Optional Step – Configure Local DNS Names Resolution**
  
- Sometimes it is tedious to remember and switch between IP addresses, especially if you have a lot of servers under your management.
What we can do, is to configure local domain name resolution. The easiest way is to use **/etc/hosts** file, although this approach is not very scalable, but it is very easy to configure and shows the concept well. So let us configure IP address to domain name mapping for our LB.
  
```
#I Opened this file on my LB server

sudo vi /etc/hosts

#Add 3 records into this file with Local IP address and arbitrary name for both of your Web Servers

Script  
<WebServer1-Private-IP-Address> Web1
<WebServer2-Private-IP-Address> Web2
<WebServer2-Private-IP-Address> Web3  
  
Executed
<WebServer1-Private-IP-Address> Web1
<WebServer2-Private-IP-Address> Web2
<WebServer2-Private-IP-Address> Web3  
```  

![8_6](https://github.com/EzeOnoky/Project-Base-Learning-8/assets/122687798/aa28b13a-d911-4938-b20b-c9c2e400f63c)
  
- Now I can update my LB config file with those names instead of IP addresses. 1st i opened the default config of the apache2 page using the `sudo vi` command
  
```
sudo vi /etc/apache2/sites-available/000-default.conf

#remove the IPs and replace it with the domain name  
BalancerMember http://Web1:80 loadfactor=5 timeout=1
BalancerMember http://Web2:80 loadfactor=5 timeout=1
BalancerMember http://Web3:80 loadfactor=5 timeout=1
```  

![8_7](https://github.com/EzeOnoky/Project-Base-Learning-8/assets/122687798/bcd3bebe-4077-45de-81e1-6e8fbf84839d)
  
- I tried to **curl** my Web Servers from LB locally and it worked...
  
```
curl http://Web1
curl http://Web2
```
  
Below is the HTML version of our web server 1, this is the same as the web page below.  

- Remember, this is only internal configuration and it is also local to your LB server, these names will neither be ‘resolvable’ from other servers internally nor from the Internet. 
  
- Targed Architecture
Now my set up looks like this:
  
![8_8](https://github.com/EzeOnoky/Project-Base-Learning-8/assets/122687798/e76b8330-8c4f-47f6-885c-a4527768929f)
  


