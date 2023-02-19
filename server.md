

# CONTINUOS INTEGRATION with JENKINS and NFS</h2>

### Configuring Apache with a LoadBAlancer and DEVOPS TOOLING WEBSITE SOLUTION with NFS

## Step 1
### Prepare NFS Server
- Create an EC2 Ubuntu VM
- create 4 EBS volume and attach it to the NFS Server (make sure the ebs volumes are in th same availability zone with NFS - server)
- SSH into the NFS server
```
    ssh -i "aws-key-pair" ubuntu@"private-ip-address"
```

- view the list of the attached disks<br>
``` 
lsblk
``` 
- Create  partions xvdf disk <br>
```
sudo gdisk /dev/xvdf
- do same for the rest of the disks <br>
- Then install the lvm2 package using the following command: <br>
```
sudo yum install lvm2`
- and then run the following command to check for available partitions: <br>
```
sudo lvmdiskscan
```

#### Create Physical Volume for xvdf1
```
sudo pvcreate /dev/xvdf1
```
- do same for the rest of the disks

##### Now we need to verify that our physical volume has been created successfully using the following command:
```
sudo pvs
```
#### Now we need to create a volume group using the vgcreate utility. We will use the 3 disks we created earlier to create a volume group called NFS-vg.
```
  sudo vgcreate nfs-vg /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1
```
### Use lvcreate utility to create 3 logical volumes. lv-opt lv-apps, and lv-logs. The lv-apps: would be used by the webservers, The lv-logs: would be used by web server logs, and the lv-opt: would be used by the Jenkins server(later in thr project)
```
sudo lvcreate -L 14G -n lv-opt nfs-vg
sudo lvcreate -L 14G -n lv-apps nfs-vg
sudo lvcreate -L 10G -n lv-logs nfs-vg
```
#### Now we need to verify that our logical volumes have been created successfully using the following command:
```
sudo lvs
```
#### Verify the entire setup
```
sudo vgdisplay -v #view complete setup - VG, PV, and LV
```  
```
sudo lsblk
```
#### Use mkfs.xfs to format the logical volumes with xfs filesystem.
```
sudo mkfs -t xfs /dev/nfs-vg/lv-opt
sudo mkfs -t xfs /dev/nfs-vg/lv-apps
sudo mkfs -t xfs /dev/nfs-vg/lv-logs
```
#### Create a directory for each of the logical volumes.
```
sudo mkdir -p /mnt/apps
sudo mkdir -p /mnt/logs
sudo mkdir -p /mnt/opt
```
#### Mount the logical volumes to the directories we created earlier.
```
sudo mount /dev/nfs-vg/lv-apps /mnt/apps
sudo mount /dev/nfs-vg/lv-logs /mnt/logs
sudo mount /dev/nfs-vg/lv-opt /mnt/opt
```
#### Verify that the logical volumes have been mounted successfully.
```
sudo df -h
```
#### Now we need to make the mount persistent. To do that, we need to edit the /etc/fstab file and add the following lines:
```
sudo blkid  (Copy the mount points UUID)
```
```
sudo nano /etc/fstab
```
(paste the content e.g like the one below)
UUID=45cd79b1-4f73-4caa-8a72-1a9e7efb3a66  /mnt/apps xfs defaults 0 0
UUID=bc2756e9-b0d8-4868-b1b3-d76e3af9614a  /mnt/logs xfs defaults 0 0
UUID=b003ab9c-57f8-40d2-8468-5cfcb2fb8b14  /mnt/opt xfs defaults 0 
    
#### Now we need to test the configurations and reload the daemon.
```
sudo mount -a
sudo systemctl daemon-reload  
```  
#### It is about time for us to install the NFS server. To do that, we need to install the nfs-utils package using the following 
```
sudo apt update
sudo apt install nfs-kernel-server
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```

#### Now we need to set up permissions that will allow our web servers to read, write and execute files on NFS:
```
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt
sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt
```  
#### Now we need to edit the /etc/exports file and add the following lines:
#### add the following:
```
/mnt/apps <NFS-Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <NFS-Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <NFS-Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
```
#### command to export
```
sudo exportfs -arv
```

#### Now we need to check the port being used by NFS and open it using Security Groups(add a new inbound Rule):
```
rpcinfo -p | grep nfs
```
#### SET UP inbound Rule for NFS server
#### Note: For the NFS server to be accessible from your client, you must also open the following ports: TCP 111, UDP 111, and UDP 2049 and NFS or TCP 2049.
Source IP for all ports can be 0.0.0.0/32  or <subnet-cidr>/32

## Step 2: - Configure the web servers
#### Create Ec2 Ubuntu Instance
- Make sure this web server and NFS server are both running the same subnet
- open port 80
- Install apache webserver
```
sudo apt install apache2
```
- check apache2 status
```
sudo systemctl status apache2
```
- Installing NFS Utility tool for the Ubuntu NFS Client webserver
```
sudo apt install nfs-common ## this utility allows Web-server to connect with the NFS server
```    
- Create a directory called /var/www/ and target the NFS servers export for apps
```
sudo mkdir /var/www (might be created already by apache)
```
```
sudo mount -t nfs -o rw,nosuid <NFS-Private-Server-IP>:/mnt/apps /var/www
```

- Verify that NFS was mounted successfully.
```
sudo df -h
```
#### Make sure that the changes will persist on the web server after reboot.
```       
sudo nano /etc/fstab
```
- add the following:
```
<Private-NFS-Server-IP>:/mnt/apps /var/www nfs defaults 0 0
```

Repeat the previous step for another 2 Web Servers.
To verify that Apache files and directories are available on the Web Server in /var/www and also on the NFS server in /mnt/apps. If you see the same files – it means NFS is mounted correctly. You can try to create a new file touch test.txt from one server and check if the same file is accessible from other Web Servers.
```
sudo touch test.txt
```

Now we need to locate the log folder for Apache on the web server and mount it to the NFS servers export for logs and make sure that the changes will persist on the web server after reboot on all the web servers.
```
sudo mkdir /var/log/apache
sudo mount -t nfs -o rw,nosuid <NFS-Server-IP>:/mnt/logs /var/log/apache
```
and then add the following to /etc/fstab:
```
sudo nano /etc/fstab
```
-add the following:
```
<NFS-Server-IP>:/mnt/logs /var/log/apache nfs defaults 0 0
```        



  <h2>OPTIONAL</h2>

<h3>CONFIGURE APACHE AS A LOAD BALANCER</h3>

STEP 1 
Create an Ubuntu Server 20.04 EC2 instance and name it lb-webserver, so your EC2 list will look like this:
Open TCP port 80 on lb-webserver by creating an Inbound Rule in Security Group.

#Install apache2
sudo apt update
sudo apt install apache2 -y
sudo apt-get install libxml2-dev

#Enable following modules:
sudo a2enmod rewrite
sudo a2enmod proxy
sudo a2enmod proxy_balancer
sudo a2enmod proxy_http
sudo a2enmod headers
sudo a2enmod lbmethod_bytraffic

#Restart apache2 service
sudo systemctl restart apache2

Make sure apache2 is up and running

Configure load balancing
sudo vi /etc/apache2/sites-available/000-default.conf


<VirtualHost *:80>  

    <Proxy "balancer://mycluster">
        BalancerMember http://<WebServer1-Private-IP-Address>:80 loadfactor=5 timeout=6
        BalancerMember http://<WebServer2-Private-IP-Address>:80 loadfactor=5 timeout=6
        ProxySet lbmethod=bytraffic
        # ProxySet lbmethod=byrequests
    </Proxy>

    ProxyPreserveHost On
    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/
</VirtualHost>



Restart apache server

sudo systemctl restart apache2

bytraffic balancing method will distribute incoming load between your Web Servers according to current traffic load. We can control in which proportion the traffic must be distributed by loadfactor parameter.

You can also study and try other methods, like: bybusyness, byrequests, heartbeat

Verify that our configuration works – try to access your LB’s public IP address or Public DNS name from your browser:
http://<Load-Balancer-Public-IP-Address-or-Public-DNS-Name>/login.php


Open two ssh/Putty consoles for both Web Servers and run following command:

sudo tail -f /var/log/httpd/access_log

Try to refresh your browser page http://<Load-Balancer-Public-IP-Address-or-Public-DNS-Name>/index.php several times and make sure that both servers receive HTTP GET requests from your LB – new records must appear in each server’s log file. The number of requests to each server will be approximately the same since we set loadfactor to the same value for both servers – it means that traffic will be disctributed evenly between them.

If you have configured everything correctly – your users will not even notice that their requests are served by more than one serve

Optional Step – Configure Local DNS Names Resolution
Sometimes it is tedious to remember and switch between IP addresses, especially if you have a lot of servers under your management.
What we can do, is to configure local domain name resolution. The easiest way is to use /etc/hosts file, although this approach is not very scalable, but it is very easy to configure and shows the concept well. So let us configure IP address to domain name mapping for our LB.

#Open this file on your LB server

sudo vi /etc/hosts

#Add 2 records into this file with Local IP address and arbitrary name for both of your Web Servers

<WebServer1-Private-IP-Address> Web1
<WebServer2-Private-IP-Address> Web2
Now you can update your LB config file with those names instead of IP addresses.

BalancerMember http://Web1:80 loadfactor=5 timeout=6
BalancerMember http://Web2:80 loadfactor=5 timeout=6
You can try to curl your Web Servers from LB locally curl http://Web1 or curl http://Web2 – it shall work.

Remember, this is only internal configuration and it is also local to your LB server, these names will neither be ‘resolvable’ from other servers internally nor from the Internet.


## Configuring Jenkins 
- Create an Ubuntu 20.04 EC2 Instance
- By default Jenkins server uses TCP port 8080 - open it by creating a new inbound rule in your EC2 security group.
- Install JDK (Java Development Kit) on the Jenkins server. Since Jenkins is a Java-based application
```
sudo apt update
sudo apt install default-jdk-headless
```
- Install Jenkins
```
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \
    /etc/apt/sources.list.d/jenkins.list'
sudo apt update
sudo apt-get install jenkins
```
- We have to make sure that Jenkins is running
```
sudo systemctl status jenkins
```
- Perform initial Jenkins setup. From your browser access.
```
http://<Jenkins-Server-Public-IP-Address>:8080
```
- Note: You will be prompted to provide a default admin password.
- To get retrieve the default admin password, run the following command.
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
- Then you will be asked which plugins to install - choose suggested plugins.
- Once plugin installation is done - create an admin user and you will get your Jenkin server address or skip to continue using the current login details

### Configure Jenkins to retrieve source code from GitHub using Webhooks.
- Here we will configure Jenkins to automatically retrieve source code from GitHub whenever there is a change in the source code. This would be done by using **Webhooks**
- Enable webhooks in your GitHub repository settings. Go to your GitHub repository and click on Settings > Webhooks > Add webhooks
- Go to Jenkins web console, click "New Item" and create a Freestyle project
- To connect your GitHub repository to Jenkins, you will need to provide its URL, just copy the URL from your GitHub repository
- In your Jenkins project, the configurations, choose Git repository, provide the URL of your GitHub repository and click on "Add" to add the credentials, so Jenkins can access your GitHub repository
- Save the configurations and let's try to run the build. Click on "Build Now" and you will see the build is successful.

You can open the build and check in "Console Output" if it has run successfully.

If so – congratulations! You have just made your very first Jenkins build!

Note: This build does not produce anything and it runs only when we trigger it manually. Let us fix it.

- In your Jenkins project, click on "Configure" your job/project and add these two configurations

- Build Triggers > click Github hook trigger for GITscm polling. Build when a change is pushed to GitHub, triggering the job from the GitHub webhook:

- Configure "Post-build Actions" to archive all the files – files resulting from a build are called "artifacts". in  the "files to archive" box type ** means all files of the build

- Now, let's go ahead and make some changes in any file in your GitHub repository (e.g. README.MD file) and push the changes to the master branch.

-We should see that a new build has been launched automatically (by webhook) and you can see its results – artifacts, saved on the Jenkins server.





How to fix HTTP ERROR 403 No valid crumb was included in the request in JENKINS
SOLUTION
There is an option in "Manage Jenkins" the "Global Security Settings" that "Enables the Compatibilty Mode for proxies". This helped with my issue.
Manage Jenkins > Configure Global Settings > CSRF Protection > Enable proxy compatibility

Note: You have now configured an automated Jenkins job that receives files from GitHub by webhook trigger (this method is considered as ‘push’ because the changes are being ‘pushed’ and file transfer is initiated by GitHub). There are also other methods: trigger one job (downstream) from another (upstream), poll GitHub periodically and others.

- By default, the artifacts are stored on Jenkins server locally
```
 ls /var/lib/jenkins/jobs/tooling_github/builds/<build_number>/archive/
```
## Configure Jenkins to copy files to the NFS server via ssh.

- Now we have our artifacts stored locally on the Jenkins server, the next step is to copy them to the NFS server to the /mnt/apps directory.

- As Jenkins is highly extendable and can be configured to do almost anything, we will need a plugin that is called "Publish over SSH" to copy files to the NFS server.

- Install "Publish over SSH" plugin. Go to Jenkins web console, click on "Manage Jenkins" > "Manage Plugins" > "Available" > "Publish over SSH" > "Install without restart"


- Configure the job/project to copy artifacts over to the NFS server. On the main dashboard select "Manage Jenkins" and choose the "Configure System" menu item. Scroll down to Publish over SSH plugin configuration section and configure it to be able to connect to your NFS server. The configuration is pretty straightforward, you just need to provide the following details:

- Provide a private key (contents of .pem file that you use to connect to NFS server via SSH/Putty) Arbitrary name Hostname – can be private IP address of your NFS server Username – ubuntu (since NFS server is based on EC2 with Ubuntu) Remote directory /mnt/apps since our Web Servers use it as a mounting point to retrieve files from the NFS server

- And then save the configurations.



Note: Test the configuration and make sure the connection returns Success. Remember, that TCP port 22 on the NFS server must be open to receive SSH connections.

- Now open the Jenkins project configuration page and add another one "Post-build Actions" to copy the artifacts to the NFS server. You are selecting the option "Send build artifacts over SSH".

- Now configure it to send all files produced by the build into our previously defined remote directory /mnt/apps. . In our case we want to copy all files and directories – so we use ** and then save the configurations.

- Now, let's go ahead and make some changes in any file in your GitHub repository (e.g. README.MD file) and push the changes to the master branch. Webhook will trigger the build and the artifacts will be copied to the NFS server.

- To make sure that the files in /mnt/apps have been udated – connect via SSH/Putty to your NFS server and check README.MD file

```
cat /mnt/apps/README.md
```

you can also check and verify the webservers to see if the files are located in the mounted directory (/var/www)

#### Now we have just implemented  Continous Integration solution using Jenkins CI



 <!-- 29a54e1aeb544ec6a63e678a981a44f6 -->