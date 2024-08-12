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

![image](https://github.com/user-attachments/assets/94a24568-c08c-4476-a4b0-ec9059688ee4)


### 2. Open TCP Port 80

Configure an Inbound Rule in the Security Group to open TCP port 80 on `Project-8-apache-lb`.

![image](https://github.com/user-attachments/assets/8c0a6b15-9d0d-4c24-872b-94c109a90a36)

### 3. Install and Configure Apache Load Balancer

#### i. Install Apache2

- login into the  instance:

```
ssh -i "my-devec2key.pem" ec2-user@18.219.148.178
```

- Update and upgrade Ubuntu:

```
sudo apt update && sudo apt upgrade
```

- Install Apache:

```
sudo apt install apache2 -y
```

- Install dependencies:

```
sudo apt-get install libxml2-dev
```
![image](https://github.com/user-attachments/assets/e047e386-df20-4852-8851-e92830ed1e71)


#### ii. Enable Required Modules

```
sudo a2enmod rewrite
sudo a2enmod proxy
sudo a2enmod proxy_balancer
sudo a2enmod proxy_http
sudo a2enmod headers
sudo a2enmod lbmethod_bytraffic
```

#### iii. Restart Apache2 Service

```
sudo systemctl restart apache2
sudo systemctl status apache2
```

![image](https://github.com/user-attachments/assets/a64f4bab-6ae2-407c-ab21-b4c9247b2ae0)

### Configure Load Balancing

#### i. Edit the `000-default.conf` File in `sites-available`

```
sudo vi /etc/apache2/sites-available/000-default.conf
```

#### ii. Add the Following Configuration Inside the `<VirtualHost *:80></VirtualHost>` Section:

```
<Proxy “balancer://mycluster”>
    BalancerMember http://172.31.46.91:80 loadfactor=5 timeout=1
    BalancerMember http://172.31.43.221:80 loadfactor=5 timeout=1
    ProxySet lbmethod=bytraffic
</Proxy>

ProxyPreserveHost on
ProxyPass / balancer://mycluster/
ProxyPassReverse / balancer://mycluster/
```
![image](https://github.com/user-attachments/assets/36620086-d82e-4b60-8652-5eff3dfbac21)


#### iii. Restart Apache

```
sudo systemctl restart apache2
```

![image](https://github.com/user-attachments/assets/42e95d8a-dbbb-42d3-9aca-ed87650e57af)

The `bytraffic` method will distribute incoming load between the web servers based on current traffic. The traffic distribution proportion is controlled by the `loadfactor` parameter. Other methods like `bybusyness`, `byrequests`, or `heartbeat` can also be used.

---

## Step 4: Verify the Configuration

### i. Access the Website via the LB's Public IP or DNS Name

![image](https://github.com/user-attachments/assets/b07996c9-6694-4d3a-953f-bbb135e6b7d0)

![image](https://github.com/user-attachments/assets/a1c975e8-ef4c-44be-99cd-8b0007e536dd)

__Note__: If `/var/log/httpd` was mounted from the Web Server to the NFS Server in a previous project, unmount it to ensure each web server has its own log directory.

### ii. Unmount the NFS Directory

- Check if the Web Server's log directory is mounted to NFS:

```
df -h
sudo umount -f /var/log/httpd
```

If the directory is busy, stop the services using it first:

```
sudo systemctl stop httpd
```

- Verify the directory is unmounted:

```
df -h
```

### iii. Monitor Access Logs on Both Web Servers

Open two SSH consoles for both web servers and run:

```
sudo tail -f /var/log/httpd/access_log
```


# Optional Step: Configure Local DNS Name Resolution

To avoid the hassle of remembering and switching between IP addresses, configure local domain name resolution using the `/etc/hosts` file. While not scalable, this approach is simple and effective for demonstration purposes.

### i. Configure IP to Domain Name Mapping for the Load Balancer

- Open the hosts file:

```
sudo vi /etc/hosts
```

- Add two records for the web servers:

![image](https://github.com/user-attachments/assets/6d786219-873c-40f9-8244-2e397aeb1be6)


### ii. Update the LB Config File with Domain Names Instead of IPs

```
sudo vi /etc/apache2/sites-available/000-default.conf
```

```
BalancerMember http://Web1:80 loadfactor=5 timeout=1
BalancerMember http://Web2:80 loadfactor=5 timeout=1
```

![image](https://github.com/user-attachments/assets/24a90797-0d9b-42ee-9f79-89003f69565c)

### iii. Test the Configuration Locally

Try to `curl` the web servers from the LB:

```
curl http://Web1
```

```
curl http://Web2
```

This configuration is internal to the LB server, and these domain names will not be resolvable from other servers or the internet.


### Conclusion

The `mod_proxy_balancer` module in Apache HTTP Server offers robust load balancing capabilities, including support for sticky sessions, health checks, and multiple load balancing algorithms. Properly configuring these features ensures that web applications achieve high availability, scalability, and reliability.
### Importance of the Project and the Need for a Load Balancer

In any web application, particularly in a multi-server environment like the "Tooling Website" project, managing traffic distribution effectively is crucial. This project, which involves setting up an Apache Load Balancer, addresses several key challenges and needs:

1. **Scalability**: As web traffic grows, a single server can quickly become overwhelmed, leading to slow response times or even downtime. A load balancer allows you to add more web servers to the architecture, ensuring that the application can handle increased traffic without degradation in performance.

2. **Reliability and High Availability**: If one web server fails, a load balancer can automatically redirect traffic to other functioning servers. This redundancy ensures that the application remains accessible to users, even if one or more servers go offline.

3. **Optimized Resource Utilization**: By distributing requests evenly across multiple servers, a load balancer ensures that no single server is overburdened while others are underutilized. This leads to more efficient use of available resources and helps maintain consistent performance.

4. **Improved User Experience**: Without a load balancer, users might need to remember and use different URLs to access different servers, which is cumbersome. A load balancer simplifies this by providing a single URL for users to access the application, regardless of which server is handling their request.

5. **Simplified Maintenance**: With a load balancer in place, individual servers can be taken offline for maintenance or updates without affecting the overall availability of the application. The load balancer simply redirects traffic to the remaining servers, ensuring uninterrupted service.

6. **Enhanced Security**: Load balancers can be configured to handle SSL termination, reducing the workload on individual web servers and centralizing the management of SSL certificates. Additionally, load balancers can act as a first line of defense against certain types of attacks by filtering out malicious traffic before it reaches the web servers.

In summary, this project is essential for creating a robust, scalable, and efficient web infrastructure. By implementing an Apache Load Balancer, we can ensure that the "Tooling Website" is well-equipped to handle current and future demands, providing users with a seamless, reliable experience while optimizing the performance and availability of the underlying web servers.
