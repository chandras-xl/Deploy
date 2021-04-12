<h1>Deploy HA setup</h1>

<h4><b>Environment Details:</h4>

Java version -
Deploy requires Java SE 8 or Java SE 11

OS - CentOS Linux release 8.1.1911 (Core)
```
1. devops-centos-3   172.16.25.133 - Deploy master -0
2. devops-centos-4   172.16.21.134 - Deploy master -1
3. devops-centos-5   172.16.29.23  - Deploy worker -0
4. devops-centos-8   172.16.29.35  - Deploy worker -1
5. devops-centos-9   172.16.16.136 - RabbitMQ
6. devops-centos-10  172.16.28.68  - Postgres DB, HAproxy LoadBalancer
```
Services required:

```
1. XL Product        - Deploy engine version 10.0.0
2. Databse           - Postgres 12
3. Messaging Queue   - RabbitMQ 3.8.8
4. Loadbalancer      - HAproxy 1.8.23 
5. Shared storage    - NFS v3 file share
```
<h1> Setup </h1>

Since this is a dev setup we will diable the firewalld service on all the nodes
```
systemctl status firewalld.service
systemctl stop firewalld.service
systemctl disable firewalld.service
```

<b>step 1: (On devops-centos-10)</b>

Installing postgres database
```
sudo dnf module enable postgresql:12
sudo dnf install postgresql-server
/usr/bin/postgresql-setup --initdb
sudo systemctl start postgresql
sudo systemctl enable postgresql
sudo systemctl status postgresql
```
Switch user to postgres

```
su - postgres
psql
```
Run the below script to create the user and role for deploy
```
CREATE USER xldeploy WITH
  NOSUPERUSER
  NOCREATEDB
  NOCREATEROLE
  ENCRYPTED PASSWORD 'xldeploy';

CREATE DATABASE xldeploy OWNER xldeploy;
```
<b>Step 2: (On devops-centos-9)

Installing RabbitMQ
```
Install EPEL
sudo yum -y install epel-release 

Download the rabbitmq installer
curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | sudo bash 
 ```
 Install and start the rabbitMQ server.
 ```
sudo yum -y install rabbitmq-server 
echo "172.16.16.136 $(hostname -s)" | sudo tee -a /etc/hosts
sudo systemctl start rabbitmq-server 
sudo systemctl enable rabbitmq-server 
sudo systemctl status rabbitmq-server 
```
Package details can be queried using rpm command with the option -qi
```
rpm -qi rabbitmq-server
```
Create rabbitmq admin user and delete the default guest user.
```
sudo rabbitmqctl add_user admin admin 
sudo rabbitmqctl set_user_tags admin administrator 
sudo rabbitmqctl set_permissions -p / admin ".*" ".*" ".*" 

sudo rabbitmqctl delete_user guest 
sudo rabbitmqctl list_users 
```
Enable the RabbitMQ management plugins by running the below command
```
sudo rabbitmq-plugins enable rabbitmq_management
```
Below plugin enables access to xldeploy
```
rabbitmq-plugins enable rabbitmq_jms_topic_exchange
```
<b> Step 3: (On deploy master-0 and deploy-master-1)

Mount the NFS share

NFS -
Mount below filesystem to be shared between both the master instances
```
mkdir -p /mnt/deploy-demo/opt/xldeploy/xl-deploy-10.0.0-server/exports/
mount -t nfs 172.16.0.45:/deploy-demo /mnt/deploy-demo/opt/xldeploy/xl-deploy-10.0.0-server/exports/
```
Add the entry in fstab
```
vi /etc/fstab
172.16.0.45:/deploy-demo /mnt/deploy-demo/opt/xl-deploy/xldeploy-10.0.0-server/exports/  nfs defaults 0 0
```
Reload the fstab
```
mount -a
```
<b>Step 4: (On devops-centos-10)
  
Installing HAproxy 
```
dnf install haproxy -y

Take backup of the HAproxy config file
mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg-bak

nano /etc/haproxy/haproxy.cfg
```
Verify your HAproxy config file with below command.
```
haproxy -c -f /etc/haproxy/haproxy.cfg
```
Start the HAProxy service.
```
systemctl start haproxy
systemctl enable haproxy
systemctl status haproxy
```
If SELinux is preventing /usr/sbin/haproxy from name_connect access on the tcp_socket port 5002 execute below command.
```
semanage port -a -t http_port_t -p tcp 5002
ausearch -c 'haproxy' --raw | audit2allow -M my-haproxy
semodule -X 300 -i my-haproxy.pp
```

<b>Step 5: (On deploy master-0) 

Setup deploy server:

Create directory under opt
```
mkdir -p /opt/xldeploy/
cd /opt/xldeploy/
```
Download the deploy installer from xebialabs distribution
```
wget --user $USER --password $PASSWORD https://dist.xebialabs.com/customer/xl-deploy/product/10.0.0/xl-deploy-10.0.0-server.zip

unzip xl-deploy-10.0.0-server.zip
```
Place the required jar files in the lib forlder.
```
cd /opt/xldeploy/xl-deploy-10.0.0-server/lib
```
Postgres jar file
```
wget https://jdbc.postgresql.org/download/postgresql-42.2.19.jar
```
RabbitMQ JMS client
```
wget https://repo1.maven.org/maven2/com/rabbitmq/jms/rabbitmq-jms/2.2.0/rabbitmq-jms-2.2.0.jar
```
RabbitMQ Java client
```
wget https://repo1.maven.org/maven2/com/rabbitmq/amqp-client/5.11.0/amqp-client-5.11.0.jar
```
Update the xl-deploy.conf file.
```
cd /opt/xldeploy/xl-deploy-10.0.0-server/conf
nano xl-deploy.conf
```

Run the setup from below path
```
cd /opt/xldeploy/xl-deploy-10.0.0-server/bin
./run.sh -setup
```
Copy the setup to master-1, worker-0 and worker-1
```
scp -r /opt/xldeploy/xl-deploy-10.0.0-server/ root@172.16.21.134:/opt/xldeploy/xl-deploy-10.0.0-server/   - master -1
scp -r /opt/xldeploy/xl-deploy-10.0.0-server/ root@172.16.29.23:/opt/xldeploy/xl-deploy-10.0.0-server/    - worker -0
scp -r /opt/xldeploy/xl-deploy-10.0.0-server/ root@172.16.29.35:/opt/xldeploy/xl-deploy-10.0.0-server/    - worker -1
```
start the master-0
```
cd /opt/xldeploy/xl-deploy-10.0.0-server/bin
./run.sh
```

<b>Step 6: (On deploy master-1)

Start the xldeploy master-1
```
cd /opt/xldeploy/xl-deploy-10.0.0-server/bin
./run.sh
```
<b>step 7: (On worker-0)

Start the worker -0
```
cd /opt/xldeploy/xl-deploy-10.0.0-server/bin
./run.sh worker -api http://172.16.28.68 -master 172.16.25.133:8180 -master 172.16.21.134:8180 -name worker-0 -hostname devops-centos-5 -port 8181
```
<b>Step 8: (On worker-1)

Start the worker -1  
```
cd /opt/xldeploy/xl-deploy-10.0.0-server/bin
./run.sh worker -api http://172.16.28.68 -master 172.16.25.133:8180 -master 172.16.21.134:8180 -name worker-1 -hostname devops-centos-8 -port 8181
```
Access the Deploy UI with below URI
```
http://devops-centos-10.xebialabs.com/
```
Access the HAProxy stats page with below URI
```
http://devops-centos-10.xebialabs.com/stats
```
Access the RabbitMQ Dashboard with below URI
```
http://devops-centos-10.xebialabs.com:7000/
```
