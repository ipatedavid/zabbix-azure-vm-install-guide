# zabbix-azure-vm-install-guide

## Hosting Zabbix server on Azure VM
Within this project we will hosting a Zabbix server on an Azure Virtual Machine so that we can monitor network performance on a home network.

First step is to create the Azure Virtual Machine, which will be running a version of Linux server.
In order to host a VM we will need to create a Virtual Network in Azure. This will allow the VM to have an IP address assigned, through which it can communicate with the Zabbix Agents on the home network later. 

### Creating the Virtual Machine.

First step with Azure services is to create a resoruce group that will contain a virtual network and the VM itself.
Create a resource group for our project: 
```bash
az group create -l southeastasia --resource-group zabbixserver 
```
Change the location accordingly, you can get a list of available ones using "az account list-locations".

Create a default VNet:
```bash
az network vnet create --resource-group zabbixserver --name default_vnet -l southeastasia
```
Create VM running Ubuntu Server: 
```bash
az vmname="ZabbixServer" username="Zabbix" az vm create --resource-group zabbixserver -n ZabbixServer --image Ubuntu2204 --public-ip-sku Standard --admin-username zabbixadmin --generate-ssh-keys
```
This will also create an SSH login for us and the respective SSH keys.

We will need the public IP of our VM to SSH into it: 
```bash
az vm list-ip-addresses -g zabbixserver -n ZabbixServer
```
Then we SSH to the VM: 
```bash
ssh zabbixadmin@[VM public IP]
```



## Installing Zabbix components and setting up the Server.

Steps on how to install Zabbix components. For my case, using an image of Ubuntu 22.04 LTS: 
```bash
wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu22.04_all.deb
dpkg -i zabbix-release_6.4-1+ubuntu22.04_all.deb
apt update
apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

Once this is done we should install MySQL Server on our machine, and assign it an admin account:
```bash
apt install mysql-server
systemctl start mysql
sudo mysql
mysql> create database zabbixserverdb character set utf8mb4 collate utf8mb4_bin;
mysql> create user dbadmin@localhost identified by 'password';
```
Replace password with a safe and secure password. 
```bash
mysql> grant all privileges on zabbixserverdb.* to dbadmin@localhost;
mysql> set global log_bin_trust_function_creators = 1;
mysql> quit;
```
Next, we need to import the database schema that our SQL database will use, from the zabbix-sql-scripts, like so:
```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -udbadmin -p zabbixserverdb
```
Once that is done, we can remove the log_bin_trust_function_creators = 1:
```bash
sudo mysql
mysql> set global log_bin_trust_function_creators = 0;
mysql> quit;
```
Now we should configure the zabbix-server.conf file to reflect all our database details, database name, user and password. Remember to change the default values if needed, for example, DBName should be "zabbixserverdb" in this case.
```bash
nano /etc/zabbix/zabbix_server.conf
```

Now that we have our database ready, we can start the apache web server and add it for the start-up processes:
```bash
sudo systemctl restart zabbix-server zabbix-agent apache2
sudo systemctl enable zabbix-server zabbix-agent apache2
```
All that is left to do is to open port 80 on our VM so that we can access the Zabbix web front from our home network. 
```bash
az vm open-port -n ZabbixServer -g zabbixserver --port 80
```
You can also update the rule to allow a specific source IP to access port 80 for added security:
```bash
az network nsg rule update --resource-group zabbixserver --nsg-name zabbixserverNSG --name open-port-80 --source-address-prefixes [your home network IP]
```
__Open the web UI at HTTP:// [hostIP] /zabbix and go through the set-up steps.__


## Setting up the Zabbix agent.

Install instructions can be found on the Zabbix download page and they are similar to the server installation, even simpler. Run the following commands:
```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-1+ubuntu22.04_all.deb
dpkg -i zabbix-release_7.0-1+ubuntu22.04_all.deb
apt update
apt install zabbix-agent
```
And to start and enable the agent daemon at system startup: 
```bash
sudo systemctl restart zabbix-agent
sudo systemctl enable zabbix-agent 
```
I've chosen to secure the connection between server and agent with a pre-shared key PSK.
Creating PSK file in zabbix folder of the agent host:
```bash
sudo -i
openssl rand -hex 32 > /etc/zabbix/PSK001.psk
```
Open the agent config file on the monitored host and edit the following:
```bash
nano /etc/zabbix/zabbix_agentd.conf
```
- Server=[IP of VM] 
- HostName=Agent001
- TLSConnect=psk
- TLSAccept=psk
- TLSPSKIdentity=PSK001
- TLSPSKFile=/etc/zabbix/PSK001.psk

__The agentd.conf settings should reflect in the web interface of the server when adding a new host.
After adding the HostName in the web portal make sure to assign a template to the host.
On the encryption page you can also provision the server with the PSK Identity and key value for our host.__ 

__Additional firewall settings should be considered for this to work.
Make sure that you have port-forwarding on your router is set-up for port 10050 to the agent machine.
And allow communication to port 10050 on the agent host machine firewall.__
```bash
sudo ufw enable
sudo ufw allow 10050 
```
You can check the results of the Uncomplicated Firewall command with:
```bash
sudo netstat -lutn | grep 10050
```
Our new rule should look something like this:
|Proto|	Recv. 	|Send	 |Local Addr.		  |Foreign Addr. |   State  |
|-----|--------|------|---------------|--------------|----------|
|tcp  |      0 |     0|  0.0.0.0:10050|  0.0.0.0:*   |   LISTEN |    
|tcp6 |      0 |     0|  :::10050     |  :::*        |   LISTEN |    




## Troubleshooting:

1. I could not get the server to communicate with the agent in passive mode, due to my home network being behind a double NAT.
Beacuse of this and because installing a VPN or Zabbix Proxy would go far beyond the scope of this guide, I decided to run the agent in active mode.

For the active agent set-up you will need to change the following in the zabbix_agentd.conf:
 - ServerActive=[IP of VM]

You will also need to open port 10051 on the VM so the agent can communicate to the server, using:
```bash
az vm open-port -n ZabbixServer -g zabbixserver --port 10051 --priority 899
```

2. Originally I have created the VM with too low specs, this resulted in the zabbix server crashing. Be aware of your VM specifications before install. In my case I had to resize the VM with:
```bash
az vm resize \
    --resource-group "zabbixserver" \
    --name ZabbixServer \
    --size Standard_DS2_v2   
```

To check the connection you can run zabbix_get commands from the server host:
```bash
zabbix_get -s agentIP -p 10050 -k agent.version
```
During my troubleshooting I got either:
- zabbix_get [3589]: Get value error: cannot connect to [[x.x.x.x]:10050]: [111] Connection refused.
- zabbix_get [3624]: Timeout while executing operation.
This was even if all the configurations were set accordingly, which prompted me to check if my agent is behind a double NAT, which it was indeed the configuration of my ISP.

Hope this helped someone, it is by far a practical ideea for home network monitoring, because of the VMs added cost. It was a good learning experience however.
