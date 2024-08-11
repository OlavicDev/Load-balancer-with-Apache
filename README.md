# Load-balancer-with-Apache

## Introduction to Load Balancer Solution with Apache

In large-scale web applications, managing traffic efficiently across multiple servers is crucial for ensuring high availability and optimal performance. In the context of the "Tooling Website" project, which involves multiple web servers, a database server, and an NFS server, the need for a streamlined, user-friendly access point becomes apparent. Rather than having users access different web servers directly, which can be cumbersome and inefficient, a load balancer offers a single point of entry.

An Apache Load Balancer sits in front of the web servers, distributing incoming traffic evenly among them. This not only simplifies access for users, providing a consistent experience regardless of which server is handling the request, but also enhances the system's scalability and fault tolerance. By configuring Apache as a load balancer on a dedicated Ubuntu EC2 instance, we can ensure that the load is balanced across the web servers, preventing any single server from becoming a bottleneck.

In this project, you will set up and configure Apache as a load balancer to distribute traffic across the existing RHEL 9 web servers, ensuring that the Tooling Website remains accessible, reliable, and performant under varying loads.


![image](https://github.com/user-attachments/assets/d17dd280-dfd5-430e-8439-5877e3f4bdde)


## Task

Deploy and configure an Apache Load Balancer for the Tooling Website solution on a dedicated Ubuntu EC2 instance. Ensure that users can access the web servers via the Load Balancer.

## Prerequisites

Before starting, ensure the following servers are already installed and configured:

- Two RHEL 9 Web Servers
- One MySQL Database Server (Ubuntu 24.04)
- One RHEL 9 NFS Server

### Prerequisite Configurations

- Apache (httpd) is running on both web servers.
- The `/var/www` directories on both web servers are mounted to `/mnt/apps` on the NFS Server.
- All necessary TCP/UDP ports are open on the Web, DB, and NFS Servers.
- Clients can access both web servers via their Public IP addresses or DNS names and open the `Tooling Website` (e.g., `http://<Public-IP-Address-or-Public-DNS-Name>/index.php`).

---

# Step 1: Configure Apache as a Load Balancer

### 1. Create an Ubuntu Server 24.04 EC2 Instance Named `Project-8-apache-lb`

![EC2 LB](./images/ec2-lb-detail.png)

### 2. Open TCP Port 80

Configure an Inbound Rule in the Security Group to open TCP port 80 on `Project-8-apache-lb`.

![Port 80](./images/lb-security-rule.png)

### 3. Install and Configure Apache Load Balancer

#### i. Install Apache2

- Access the instance:

```bash
ssh -i "my-devec2key.pem" ec2-user@18.219.148.178
```

![SSH](./images/ssh-lb.png)

- Update and upgrade Ubuntu:

```bash
sudo apt update && sudo apt upgrade
```

![Update Ubuntu](./images/update-upgrade-lb.png)

- Install Apache:

```bash
sudo apt install apache2 -y
```

![Apache](./images/install-apache.png)

- Install dependencies:

```bash
sudo apt-get install libxml2-dev
```

![Apache Dependencies](./images/install-dependencies.png)

#### ii. Enable Required Modules

```bash
sudo a2enmod rewrite
sudo a2enmod proxy
sudo a2enmod proxy_balancer
sudo a2enmod proxy_http
sudo a2enmod headers
sudo a2enmod lbmethod_bytraffic
```

![Modules](./images/enable-modules.png)

#### iii. Restart Apache2 Service

```bash
sudo systemctl restart apache2
sudo systemctl status apache2
```

![Restart Apache](./images/restart-apache.png)

### Configure Load Balancing

#### i. Edit the `000-default.conf` File in `sites-available`

```bash
sudo vi /etc/apache2/sites-available/000-default.conf
```

#### ii. Add the Following Configuration Inside the `<VirtualHost *:80></VirtualHost>` Section:

```apache
<Proxy “balancer://mycluster”>
    BalancerMember http://172.31.46.91:80 loadfactor=5 timeout=1
    BalancerMember http://172.31.43.221:80 loadfactor=5 timeout=1
    ProxySet lbmethod=bytraffic
</Proxy>

ProxyPreserveHost on
ProxyPass / balancer://mycluster/
ProxyPassReverse / balancer://mycluster/
```

![Server Config](./images/apache-server-config.png)

#### iii. Restart Apache

```bash
sudo systemctl restart apache2
```

![Restart Apache Status](./images/restart-apache-status.png)

The `bytraffic` method will distribute incoming load between the web servers based on current traffic. The traffic distribution proportion is controlled by the `loadfactor` parameter. Other methods like `bybusyness`, `byrequests`, or `heartbeat` can also be used.

---

## Step 4: Verify the Configuration

### i. Access the Website via the LB's Public IP or DNS Name

![LB Public IP](./images/lb-public-ip.png)

![LB Website](./images/lb-wesite.png)

__Note__: If `/var/log/httpd` was mounted from the Web Server to the NFS Server in a previous project, unmount it to ensure each web server has its own log directory.

### ii. Unmount the NFS Directory

- Check if the Web Server's log directory is mounted to NFS:

```bash
df -h
sudo umount -f /var/log/httpd
```

If the directory is busy, stop the services using it first:

```bash
sudo systemctl stop httpd
```

- Verify the directory is unmounted:

```bash
df -h
```

![Unmount](./images/unmount.png)

### iii. Monitor Access Logs on Both Web Servers

Open two SSH consoles for both web servers and run:

```bash
sudo tail -f /var/log/httpd/access_log
```

Web Server 1 Access Log:

![Web1 Access Log](./images/access-log-web1-1.png)

Web Server 2 Access Log:

![Web2 Access Log](./images/access-log-web2-1.png)

### iv. Refresh the Browser Page Several Times

Ensure both web servers receive HTTP GET requests. New records should appear in each web server’s log files. Since `loadfactor` is set to the same value for both servers, traffic will be evenly distributed between them.

Web Server 1 Access Log:

![Logs](./images/access-log-web1-2.png)

Web Server 2 Access Log:

![Logs](./images/access-log-web2-2.png)

---

# Optional Step: Configure Local DNS Name Resolution

To avoid the hassle of remembering and switching between IP addresses, configure local domain name resolution using the `/etc/hosts` file. While not scalable, this approach is simple and effective for demonstration purposes.

### i. Configure IP to Domain Name Mapping for the Load Balancer

- Open the hosts file:

```bash
sudo vi /etc/hosts
```

- Add two records for the web servers:

![DNS Hosts](./images/dns-hosts.png)

### ii. Update the LB Config File with Domain Names Instead of IPs

```bash
sudo vi /etc/apache2/sites-available/000-default.conf
```

```apache
BalancerMember http://Web1:80 loadfactor=5 timeout=1
BalancerMember http://Web2:80 loadfactor=5 timeout=1
```

![DNS Name](./images/dns-apache-config.png)

### iii. Test the Configuration Locally

Try to `curl` the web servers from the LB:

```bash
curl http://Web1
```

![Curl Web1](./images/curl-web1.png)

```bash
curl http://Web2
```

![Curl Web2](./images/curl-web2.png)

This configuration is internal to the LB server, and these domain names will not be resolvable from other servers or the internet.


### Conclusion

The `mod_proxy_balancer` module in Apache HTTP Server provides powerful features for load balancing, including sticky sessions, health checks, and various load balancing algorithms. Proper configuration of these options ensures high availability, scalability, and reliability for web applications.
