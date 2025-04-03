# Installation of IBM Process Mining V2.0 - Traditional on RHEL

> Disclaimer: This document does not replace the official IBM Process Mining documentation which is available at https://www.ibm.com/docs/en/process-mining/. Use the documentation for planning and executing the installation.

## Installation information
- Date of installation: March 26, 2025
- Operating System: Red Hat Enterprise Linux release 9.2 (Plow)

## Deployment architecture
* All components running in one VM.
* MongoDB - Installed manually as a pre-requisite as if it installed on a dedicated server, connected using a network connection from Process Mining. The network connection connect Process Mining and MongoDB, there are no local files dependencies.
* MonetDB - Using bundled version with PM. Connecting PM remotely to MonetDB is not officially supported yet. TBD follow-up on IBM Support Case TS018813066 trying to get statement about future roadmap.
* Process Mining - Installed under non-root user into /opt/processmining

IBM Process Mining - Installation planning documentation: https://www.ibm.com/docs/en/process-mining/2.0.0?topic=environments-planning-installation

IBM Process Mining - System requirements documentation: https://www.ibm.com/docs/en/process-mining/2.0.0?topic=installation-system-requirements


## Supported databases:
- MongoDB 6.0 or 7.0 - Using the latest sub-version of version 7 following statement from IBM Support below. Currently version 7.0.18. Using Community Edition.

> Statement from IBM Support on MongoDB versions: We do specify versions 6.0 or 7.0 https://www.ibm.com/docs/en/process-mining/2.0.0?topic=installation-system-requirements#software-requirement
As these have been tested with Process Mining.
Now there is more flexibility on version choice for  MongoDB.
However MongoDB themselves list End of life dates for older versions https://www.mongodb.com/legal/support-policy/lifecycles
As you can see only version 6 & 7 are still active, and Version 8 is just out.

- MonetDB 11.49.11 - Using bundled installation with PM.

> Statement from IBM Support on MonetDB versions: MonetDB is "bundled" within 2.0 of Process Mining.
you do not need to install it manually.
as stated
https://www.ibm.com/docs/en/process-mining/2.0.0?topic=integration-database#installing-monetDB

## Installation procedure - MongoDB V7 Community Edition

Manual installation of MongoDB is a pre-requisite for Process Mining.

Following:
- https://www.ibm.com/docs/en/process-mining/2.0.0?topic=integration-database#installing-mongodb-instance
- https://www.mongodb.com/docs/v7.0/tutorial/install-mongodb-on-red-hat/

Installing using yum as root.

```
# Switch to root
sudo su -

# Add yum repo for MongoDB V7
cat > /etc/yum.repos.d/mongodb-org-7.0.repo <<EOF
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/9/mongodb-org/7.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://pgp.mongodb.com/server-7.0.asc
EOF

# Install the latest sub-version of version 7
sudo yum install -y mongodb-org

# Swap mongosh binary to support OpenSSL, otherwise SSL error appears when running mongosh
sudo yum swap mongodb-mongosh mongodb-mongosh-shared-openssl3

# Add MongoDB to systemd
sudo systemctl start mongod
sudo systemctl daemon-reload
sudo systemctl enable mongod
sudo systemctl status mongod

# Optional - Defensively prevent MongoDB automatic upgrades by package pinning
# DELETE THIS IF PLAYING AROUND WITH MONGO PACKAGES E.G. BEFORE MANUAL UPGRADE
cat >> /etc/yum.conf <<EOF

# Defensively prevent MongoDB automatic upgrades by package pinning
exclude=mongodb-org,mongodb-org-database,mongodb-org-server,mongodb-mongosh,mongodb-org-mongos,mongodb-org-tools
EOF

# Exit root shell
exit
```

### Enable MongoDB access control, add users, enable network connections

Please cross-check the setup with your internal security policies.

Start mongosh and add admin user.
```
mongosh
```

Add admin user
```
use admin

db.createUser({
  user: "admin",
  pwd: "X9X7Q^!YES2Xhi8@",
  roles: [
    {
      role: "userAdminAnyDatabase",
      db: "admin"
    },
    "readWriteAnyDatabase"
  ]
})

exit
```

Edit as root file /etc/mongod.conf
```
sudo vi /etc/mongod.conf
```
Enable security section and add content in it.
```
security:
  authorization: enabled
```
Bind MonngoDB to all network interfaces
```
net:
  port: 27017
  bindIp: 0.0.0.0 # Default is 127.0.0.1
```
Restart MongoDB
```
sudo systemctl restart mongod
```
Connect to MongoDB as admin user:
```
mongosh -u "admin" --authenticationDatabase "admin"
```
Create database processmining with user processmining to be used by Process Mining to store its data in MongoDB.
```
use processmining

db.createUser({
  user: "processmining",
  pwd: "ProcessMining123",
  roles: [
    { role: "readWrite", db: "myDatabase" }
  ]
})
```
Tune RHEL according with https://www.mongodb.com/docs/manual/administration/production-checklist-operations/ - cross-check on you own against your preferences and default setup.

## Installation procedure - Process Mining
Following installation instructions at https://www.ibm.com/docs/en/process-mining/2.0.0?topic=installing-traditional-environments

Download tgz containing the installation packages as described at https://www.ibm.com/docs/en/process-mining/2.0.0?topic=installation-process-mining-packages, part number M0NY1ML.

Unpack the installation package.
```
tar xvfz <installation-package.tgz>
cd M0NY1ML/Process\ Mining\ 2.0\ Server\ Multiplatform\ Multilingual/
```

Extracted files should like like follows:
```
License
ibmprocessmining-setup-2.0_64b80ae0.tar.gz
ibmprocessmining-update-2.0_64b80ae0.tar.gz
taskminer_setup_2.0_025c5931.tar.gz
taskminer_update_2.0_025c5931.tar.gz
```

Following the basic setup instructions at https://www.ibm.com/docs/en/process-mining/2.0.0?topic=mining-basic-setup

Set installation variables and keep them active during the whole installation in your bash!
```
export PM_HOME="/opt/processmining"
export PM_USER=itzuser
export PM_GROUP=itzuser
export CERT_DIR="/opt/cert"
export TMPDIR="/opt/processmining/repository/temp"
```
Extract the Process Mining tgz to its destination.
```
sudo tar xvfz ibmprocessmining-setup-2.0_64b80ae0.tar.gz -C /opt
sudo chown -R ${PM_USER}:${PM_GROUP} ${PM_HOME}
```
Extract and apply the Process Mining update ONLY if upgrading from '1.15.0' to '2.0'.
```
### NOT NEEDED
# tar xvfz ibmprocessmining-update-2.0_64b80ae0.tar.gz
# cd processmining-update
# bash update.sh
```
Update Process Mining configuration in file `<PM_HOME>/etc/processmining.conf`
- Hostname
```
#Complete process mining url starting with https://
processmining.host: "https://<HOSTNAME_FQDN>"
```
- MongoDB connection - encrypt the password first to have encrypted value for the config file as described at https://www.ibm.com/docs/en/process-mining/2.0.0?topic=mining-advanced-configuration-integration#password-encryption:
```
cd ${PM_HOME}/crypto-utils # Working directory here
.crypt-utils.bat <MONGODB_UNENCRYPTED_PASSWORD>
# like
./crypt-utils.sh ProcessMining123
```
Keep the `Encrypted String` like `nJCNPynMntYisSD4enuNRFC8JarqmRUoCj+Ea5Tszsc=`

Continue editing configuration file `<PM_HOME>/etc/processmining.conf`, use your database, host, port, user, encrypted password:
```
...
persistence: {
  mongodb: {
    database: "processmining",
    host: "itzvsi-060001ps1p-0959rrek",
    port: 27017,
    user: "processmining",
    password: "nJCNPynMntYisSD4enuNRFC8JarqmRUoCj+Ea5Tszsc=",
...
```

RHEL v9 has by default Python V3.9 installed in `/usr/bin/python*` which does not need any additional configuration. Check https://www.ibm.com/docs/en/process-mining/2.0.0?topic=mining-basic-setup#non-standard-python-deployment-for-custom-process-app in case yiu want to use different version of Python.

Create self-signed certificates following https://www.ibm.com/docs/en/process-mining/2.0.0?topic=installation-self-certificates before proceeding with web server configuration

Determine your server hostname with FQDN like:
```
itzvsi-060001ps1p-0959rrek.demo.ibm.cloud.com
```
Create a directory for your certs and create file
```
sudo mkdir ${CERT_DIR}
sudo chown -R ${PM_USER}:${PM_GROUP} ${CERT_DIR}
cd ${CERT_DIR}
```
Create CA private key (note password used will be: changeit)
```
openssl genrsa -des3 -out rootCA.key 2048
```

Generate CA certificate
```
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.pem
```
Insert info like
```
Country Name (2 letter code) [XX]:CZ
State or Province Name (full name) []:Czechia
Locality Name (eg, city) [Default City]:Prague
Organization Name (eg, company) [Default Company Ltd]:IBM
Organizational Unit Name (eg, section) []:SW
Common Name (eg, your name or your server's hostname) []:Radek
Email Address []:
```
Create a new file v3.ext 
```
vi v3.ext
```
Add the following text to the file.
Note that the hostname wildcard should match your server hostname.
```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = *.ibmcloud.com
```

Create the cert
```
openssl req -new -nodes -out server.csr -newkey rsa:2048 -keyout server.key

openssl x509 -req -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out server.crt -days 500 -sha256 -extfile v3.ext
```

Combine the files:
```
cat server.crt server.key > server.pem
```

Your files in the directory should look like this
```
-rw-------. 1 itzuser itzuser 1854 Mar 26 16:31 rootCA.key
-rw-------. 1 itzuser itzuser 1306 Mar 26 16:34 rootCA.pem
-rw-------. 1 itzuser itzuser   41 Mar 26 16:40 rootCA.srl
-rw-------. 1 itzuser itzuser 1350 Mar 26 16:40 server.crt
-rw-------. 1 itzuser itzuser 1021 Mar 26 16:39 server.csr
-rw-------. 1 itzuser itzuser 1704 Mar 26 16:38 server.key
-rw-------. 1 itzuser itzuser 3054 Mar 26 16:42 server.pem
-rw-------. 1 itzuser itzuser  205 Mar 26 16:37 v3.ext
```

Now go back to basic setup and proceed with Web and SSL configuration: https://www.ibm.com/docs/en/process-mining/2.0.0?topic=mining-basic-setup#web-and-ssl-configuration

Install nginx
```
sudo yum install nginx
```
Backup the original nginx virtual host template file and overwrite it with a file coming with Process Mining:
```
sudo mv /etc/nginx/conf.d/default.conf /etc/nginx/confd/default_origin.conf # The directory is empty, TODO update documentation
sudo cp ${PM_HOME}/nginx/processmining.conf /etc/nginx/conf.d/default.conf
```
Create the nginx ssl directory and copy the certs
```
sudo mkdir /etc/nginx/ssl
sudo cp /opt/cert/server.* /etc/nginx/ssl/ # For self-signed certs as in the docs
```
Update permissions - worked only in an interactive root shell after like `sudo su -`
```
sudo chcon -t httpd_config_t /etc/nginx/ssl/*.*
```
Update the nginx config to use our certs
```
sudo vi /etc/nginx/conf.d/default.conf
```
Around line 26 update the certificat files
```
ssl_certificate /etc/nginx/ssl/server.pem;
ssl_certificate_key /etc/nginx/ssl/server.key;
```
Enable and start nginx
```
sudo systemctl enable nginx
sudo systemctl start nginx
sudo setsebool -P httpd_can_network_connect 1
```
Process Mining startup
```
cd ${PM_HOME}/bin
```
Create start script
```
vi start.sh
```

Add following commands
```
cd /opt/processmining/bin
./pm-monet.sh start
./pm-web.sh start
./pm-engine.sh start
./pm-analytics.sh start
./pm-accelerators.sh start
./pm-monitoring.sh start
```
Create stop script
```
vi stop.sh
```
Add following commands
```
cd /opt/processmining/bin
./pm-monitoring.sh stop
./pm-accelerators.sh stop
./pm-analytics.sh stop
./pm-engine.sh stop
./pm-web.sh stop
./pm-monet.sh stop
```
Make scripts executable
```
chmod +x start.sh
chmod +x stop.sh
```
Start Process Mining
```
${PM_HOME}/bin/start.sh
```
Stop Process Mining
```
${PM_HOME}/bin/stop.sh
```

### Validation of the installation
https://www.ibm.com/docs/en/process-mining/2.0.0?topic=environments-validating-installation

### Advanced setup

Check https://www.ibm.com/docs/en/process-mining/2.0.0?topic=integration-advanced-setup#running-as-a-service

Running as a service: https://www.ibm.com/docs/en/process-mining/2.0.0?topic=integration-advanced-setup#running-as-a-service


## Open points, TODOs
- Process App Accelerator - Nice to have
- Web SSL Certificates - Self-signed certificates should be replaced with certificates from customer's CA.
- MongoDB - Enable SSL connection
- MongoDB configuration for process app - https://www.ibm.com/docs/en/process-mining/2.0.0?topic=mining-basic-setup#mongodb-configuration-for-process-app

## Misc - NOT USED

### Access MongoDB using mongosh
```
mongosh "mongodb://localhost:27017/processmining" -u "processmining" -p "ProcessMining123"

```

### Uninstall manually installed MonetDB

```
sudo yum remove MonetDB-client
sudo yum remove MonetDB-SQL-server5
sudo yum remove MonetDB-release-epel.noarch
sudo yum list installed | grep -i monet # output should be empty
```

Uninstall manually installed MongoDB
```
sudo systemctl stop mongod
sudo yum erase $(rpm -qa | grep mongodb-org)
sudo rm -r /var/log/mongodb
sudo rm -r /var/lib/mongo
```

### MonetDB - own installation not using the bundled installation

Update crypto policies to be able to import MonetDB GPG Key - not in IBM Docs
```
update-crypto-policies --set DEFAULT:SHA1
```
Install MonetDB packages
```
sudo rpm --import https://dev.monetdb.org/downloads/MonetDB-GPG-KEY
sudo yum install https://dev.monetdb.org/downloads/epel/MonetDB-release-epel.noarch.rpm
sudo yum install MonetDB-SQL-server5 MonetDB-client
```

Add PM user to the MonetDB group - not in IBM Docs
```
sudo usermod -a -G monetdb $PM_USER
```

Ensure that MonetDB is not enabled with the following command on Ubuntu and RedHat:
```
sudo systemctl status monetdbd
sudo systemctl stop monetdbd
sudo systemctl disable monetdbd
```

Run the following commands to enable MonetDB the access to the library file shipped with IBM Process Mining:

```
pushd /usr/lib64/monetdb5-11.51.7 # The directory according with the MonetDB version
sudo cp ${PM_HOME}/native/linux/x64/*.so .
sudo ln -s lib_PM-sql-2.0.so lib_PM-sql.so
popd
```

In order to use alternative MonetDB binaries, you must set override in <PM_HOME>/bin/environment.conf:

```
PM_MONET_OVERRIDE=1
...
PM_MONET_BINARIES_FOLDER=/usr/bin
...
PM_MONET_LIB_FOLDER=/usr/lib64/monetdb5-11.51.7
```
