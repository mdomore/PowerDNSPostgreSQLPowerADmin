## PowerDNSPostgreSQLPowerADmin
Installation procedure of PowerDNS with backend PostgreSQL and PowerDNS-Admin on Ubuntu 16.04

## Table of Contents
[Install Ubuntu 16.04](https://github.com/mdomore/PowerDNSPostgreSQLPowerADmin/blob/master/README.md#install-ubuntu-1604)

[Install PowerDNS & PostgreSQL backend](https://github.com/mdomore/PowerDNSPostgreSQLPowerADmin/blob/master/README.md#install-powerdns--postgresql-backend)

[Install PostgreSQL](https://github.com/mdomore/PowerDNSPostgreSQLPowerADmin/blob/master/README.md#install-postgresql)

[Install Powerdns](https://github.com/mdomore/PowerDNSPostgreSQLPowerADmin/blob/master/README.md#install-powerdns)

[Configure PowerDNS & PostgreSQL backend](https://github.com/mdomore/PowerDNSPostgreSQLPowerADmin/blob/master/README.md#configure-powerdns--postgresql-backend)

[Configure PostegreSQL](https://github.com/mdomore/PowerDNSPostgreSQLPowerADmin/blob/master/README.md#configure-postegresql)

[Restart Services and check status](https://github.com/mdomore/PowerDNSPostgreSQLPowerADmin/blob/master/README.md#restart-services-and-check-status)

[Test serveur](https://github.com/mdomore/PowerDNSPostgreSQLPowerADmin/blob/master/README.md#test-serveur)

[Install WebUi PowerDNS-Admin](https://github.com/mdomore/PowerDNSPostgreSQLPowerADmin/blob/master/README.md#install-webui-powerdns-admin)

[Start the webui with systemd](https://github.com/mdomore/PowerDNSPostgreSQLPowerADmin/blob/master/README.md#start-the-webui-with-systemd)

[Access Webui](https://github.com/mdomore/PowerDNSPostgreSQLPowerADmin/blob/master/README.md#access-webui)

[Configure Master & Slave for DNS replication](https://github.com/mdomore/PowerDNSPostgreSQLPowerADmin/blob/master/README.md#configure-master--slave-for-dns-replication)

[Load zone from Bind](https://github.com/mdomore/PowerDNSPostgreSQLPowerADmin/blob/master/README.md#load-zone-from-bind)

[For master server]()

[For slave server]()


## Install Ubuntu 16.04
### Install and upgrade ubuntu
```
sudo apt-get update && sudo apt-get upgrade
```
## Install PowerDNS & PostgreSQL backend
### Install Postgresql
```
sudo apt-get install postgresql postgresql-contrib
```
### Install PowerDNS
Install Powerdns 4.0.X
You can get info at : https://repo.powerdns.com/
Create the file '/etc/apt/sources.list.d/pdns.list' with this content:
```
deb [arch=amd64] http://repo.powerdns.com/ubuntu xenial-auth-40 main
```

And this to '/etc/apt/preferences.d/pdns':
```
Package: pdns-*
Pin: origin repo.powerdns.com
Pin-Priority: 600
```

and execute the following commands:
```
curl https://repo.powerdns.com/FD380FBB-pub.asc | sudo apt-key add - &&
sudo apt-get update &&
sudo apt-get install pdns-server pdns-backend-pgsql
```
## Configure PowerDNS & PostgreSQL backend
### Configure PostegreSQL

Create user
```
sudo -u postgres createuser --interactive
Enter name of role to add: pdns
Shall the new role be a superuser? (y/n) n
Shall the new role be allowed to create databases? (y/n) y
Shall the new role be allowed to create more new roles? (y/n) n
```

Create database
```
sudo -u postgres createdb pdns
```

Set password
```
sudo -u pdns psql
pdns=> \password pdns
```

Populate database for PowerDNS
```
CREATE TABLE domains (
  id                    SERIAL PRIMARY KEY,
  name                  VARCHAR(255) NOT NULL,
  master                VARCHAR(128) DEFAULT NULL,
  last_check            INT DEFAULT NULL,
  type                  VARCHAR(6) NOT NULL,
  notified_serial       INT DEFAULT NULL,
  account               VARCHAR(40) DEFAULT NULL,
  CONSTRAINT c_lowercase_name CHECK (((name)::TEXT = LOWER((name)::TEXT)))
);

CREATE UNIQUE INDEX name_index ON domains(name);


CREATE TABLE records (
  id                    SERIAL PRIMARY KEY,
  domain_id             INT DEFAULT NULL,
  name                  VARCHAR(255) DEFAULT NULL,
  type                  VARCHAR(10) DEFAULT NULL,
  content               VARCHAR(65535) DEFAULT NULL,
  ttl                   INT DEFAULT NULL,
  prio                  INT DEFAULT NULL,
  change_date           INT DEFAULT NULL,
  disabled              BOOL DEFAULT 'f',
  ordername             VARCHAR(255),
  auth                  BOOL DEFAULT 't',
  CONSTRAINT domain_exists
  FOREIGN KEY(domain_id) REFERENCES domains(id)
  ON DELETE CASCADE,
  CONSTRAINT c_lowercase_name CHECK (((name)::TEXT = LOWER((name)::TEXT)))
);

CREATE INDEX rec_name_index ON records(name);
CREATE INDEX nametype_index ON records(name,type);
CREATE INDEX domain_id ON records(domain_id);
CREATE INDEX recordorder ON records (domain_id, ordername text_pattern_ops);


CREATE TABLE supermasters (
  ip                    INET NOT NULL,
  nameserver            VARCHAR(255) NOT NULL,
  account               VARCHAR(40) NOT NULL,
  PRIMARY KEY(ip, nameserver)
);


CREATE TABLE comments (
  id                    SERIAL PRIMARY KEY,
  domain_id             INT NOT NULL,
  name                  VARCHAR(255) NOT NULL,
  type                  VARCHAR(10) NOT NULL,
  modified_at           INT NOT NULL,
  account               VARCHAR(40) DEFAULT NULL,
  comment               VARCHAR(65535) NOT NULL,
  CONSTRAINT domain_exists
  FOREIGN KEY(domain_id) REFERENCES domains(id)
  ON DELETE CASCADE,
  CONSTRAINT c_lowercase_name CHECK (((name)::TEXT = LOWER((name)::TEXT)))
);

CREATE INDEX comments_domain_id_idx ON comments (domain_id);
CREATE INDEX comments_name_type_idx ON comments (name, type);
CREATE INDEX comments_order_idx ON comments (domain_id, modified_at);


CREATE TABLE domainmetadata (
  id                    SERIAL PRIMARY KEY,
  domain_id             INT REFERENCES domains(id) ON DELETE CASCADE,
  kind                  VARCHAR(32),
  content               TEXT
);

CREATE INDEX domainidmetaindex ON domainmetadata(domain_id);


CREATE TABLE cryptokeys (
  id                    SERIAL PRIMARY KEY,
  domain_id             INT REFERENCES domains(id) ON DELETE CASCADE,
  flags                 INT NOT NULL,
  active                BOOL,
  content               TEXT
);

CREATE INDEX domainidindex ON cryptokeys(domain_id);


CREATE TABLE tsigkeys (
  id                    SERIAL PRIMARY KEY,
  name                  VARCHAR(255),
  algorithm             VARCHAR(50),
  secret                VARCHAR(255),
  CONSTRAINT c_lowercase_name CHECK (((name)::TEXT = LOWER((name)::TEXT)))
);

CREATE UNIQUE INDEX namealgoindex ON tsigkeys(name, algorithm);
```

Change authorisation details
Edit the file ‘/etc/postgresql/9.5/main/pg_hba.conf’ with the content
```
# "local" is for Unix domain socket connections only
local   pdns            pdns                                    md5
local   all             all                                     peer
# IPv4 local connections:
host    pdns            pdns            127.0.0.1/32            md5
host    all             all             127.0.0.1/32            md5
```

Configure Powerdns
Edit the file ‘/etc/powerdns/pdns.conf’ with the content
```
api=yes
api-key=<somekey>
launch=gpgsql
webserver=yes
master=yes
allow-axfr-ips=<slave_server_ip>/32
also-notify=<slave_server_ip>
```

Create the file ‘/etc/powerdns/pdns.d/pdns.gpgsql.conf’ with the content
```
launch=gpgsql
gpgsql-host=/run/postgresql # if PostgreSQL is listening to unix socket
#gpgsql-host=127.0.0.1
#gpgsql-port=5432
gpgsql-dbname=pdns
gpgsql-user=pdns
gpgsql-password=<password>
```

## Restart Services and check status
```
systemctl restart postgresql
systemctl restart pdns
systemctl status postgresql
```
```
● postgresql.service - PostgreSQL RDBMS
   Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
   Active: active (exited) since Fri 2017-06-30 12:17:32 CEST; 38s ago
  Process: 32699 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
 Main PID: 32699 (code=exited, status=0/SUCCESS)
    Tasks: 0
   Memory: 0B
      CPU: 0
   CGroup: /system.slice/postgresql.service
```
```
systemctl status pdns
```
```
● pdns.service - PowerDNS Authoritative Server
   Loaded: loaded (/lib/systemd/system/pdns.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2017-06-30 12:17:40 CEST; 35s ago
     Docs: man:pdns_server(1)
           man:pdns_control(1)
           https://doc.powerdns.com
 Main PID: 32715 (pdns_server)
    Tasks: 10
   Memory: 5.2M
      CPU: 21ms
   CGroup: /system.slice/pdns.service
           └─32715 /usr/sbin/pdns_server --guardian=no --daemon=no --disable-syslog --write-pid=no
```
## Test serveur
dig chaos txt version.bind @127.0.0.1 +short
```
"PowerDNS Authoritative Server 4.0.4 (built Jun 22 2017 20:08:59 by root@12d54a62098e)"
```

## Install WebUi PowerDNS-Admin
Get source from github
```
git clone https://github.com/ngoduykhanh/PowerDNS-Admin.git
```

Install dependency
```
sudo apt-get install postgresql-server-dev-all python-pip libsasl2-dev python-dev libldap2-dev libssl-dev
```
```
pip install --upgrade pip
```
```
sudo pip install psycopg2
```
```
cd PowerDNS-Admin
sudo -H pip install -r requirements.txt
```
Copy file ‘config_template.py’ to ‘config.py’ and fil it with this content
```
import os
basedir = os.path.abspath(os.path.dirname(__file__))

# BASIC APP CONFIG
WTF_CSRF_ENABLED = True
SECRET_KEY = '<somekey>'
BIND_ADDRESS = '<server_ip>'
PORT = 9393
LOGIN_TITLE = "PDNS"

# TIMEOUT - for large zones
TIMEOUT = 10

# LOG CONFIG
LOG_LEVEL = 'DEBUG'
LOG_FILE = 'logfile.log'
# For Docker, leave empty string
#LOG_FILE = ''

# Upload
UPLOAD_DIR = os.path.join(basedir, 'upload')

# DATABASE CONFIG
SQLA_DB_USER = 'pdns'
SQLA_DB_PASSWORD = 'password'
SQLA_DB_HOST = '127.0.0.1'
SQLA_DB_NAME = 'pdns'

#MySQL
#PGSQL
SQLALCHEMY_DATABASE_URI = 'postgresql://'+SQLA_DB_USER+':'\
    +SQLA_DB_PASSWORD+'@'+SQLA_DB_HOST+'/'+SQLA_DB_NAME
SQLALCHEMY_MIGRATE_REPO = os.path.join(basedir, 'db_repository')
SQLALCHEMY_TRACK_MODIFICATIONS = True

# LDAP CONFIG
LDAP_TYPE = 'ldap'
LDAP_URI = 'ldaps://your-ldap-server:636'
LDAP_USERNAME = 'cn=dnsuser,ou=users,ou=services,dc=duykhanh,dc=me'
LDAP_PASSWORD = 'dnsuser'
LDAP_SEARCH_BASE = 'ou=System Admins,ou=People,dc=duykhanh,dc=me'
# Additional options only if LDAP_TYPE=ldap
LDAP_USERNAMEFIELD = 'uid'
LDAP_FILTER = '(objectClass=inetorgperson)'

## AD CONFIG
#LDAP_TYPE = 'ad'
#LDAP_URI = 'ldaps://your-ad-server:636'
#LDAP_USERNAME = 'cn=dnsuser,ou=Users,dc=domain,dc=local'
#LDAP_PASSWORD = 'dnsuser'
#LDAP_SEARCH_BASE = 'dc=domain,dc=local'
## You may prefer 'userPrincipalName' instead
#LDAP_USERNAMEFIELD = 'sAMAccountName'
## AD Group that you would like to have accesss to web app
#LDAP_FILTER = 'memberof=cn=DNS_users,ou=Groups,dc=domain,dc=local'

# Github Oauth
GITHUB_OAUTH_ENABLE = False
GITHUB_OAUTH_KEY = 'G0j1Q15aRsn36B3aD6nwKLiYbeirrUPU8nDd1wOC'
GITHUB_OAUTH_SECRET = '0WYrKWePeBDkxlezzhFbDn1PBnCwEa0vCwVFvy6iLtgePlpT7WfUlAa9sZgm'
GITHUB_OAUTH_SCOPE = 'email'
GITHUB_OAUTH_URL = 'http://127.0.0.1:5000/api/v3/'
GITHUB_OAUTH_TOKEN = 'http://127.0.0.1:5000/oauth/token'
GITHUB_OAUTH_AUTHORIZE = 'http://127.0.0.1:5000/oauth/authorize'

#Default Auth
BASIC_ENABLED = True
SIGNUP_ENABLED = True

# POWERDNS CONFIG
PDNS_STATS_URL = 'http://127.0.0.1:8081/'
PDNS_API_KEY = '<somekey>'
PDNS_VERSION = '4.0.4'

# RECORDS ALLOWED TO EDIT
RECORDS_ALLOW_EDIT = ['A', 'AAAA', 'CNAME', 'SPF', 'PTR', 'MX', 'TXT', 'NS']

# EXPERIMENTAL FEATURES
PRETTY_IPV6_PTR = False
```

Create database :
```
sudo ./create_db.py
```

## Start the webui with systemd
Create unit file
Create a unit file ‘/lib/systemd/system/powerdnsadmin.service’ with this content :
```
[Unit]
Description=PowerDNS-Admin Service
After=multi-user.target

[Service]
Type=idle
ExecStart=/usr/bin/python /home/exploit/PowerDNS-Admin/run.py > /home/exploit/PowerDNS-Admin/run.log 2>&1

[Install]
WantedBy=multi-user.target
```

Set the permission on the unit file : 
```
sudo chmod 644 /lib/systemd/system/powerdnsadmin.service
```
Configure systemd
Now the unit file has been defined we can tell systemd to start it during the boot sequence :
```
sudo systemctl daemon-reload
sudo systemctl enable powerdnsadmin.service
```
Start the service
```
sudo systemctl start powerdnsadmin.service
```
Check status of your service

## Access Webui
To start the webui run the script ‘run.py’
You can now go to ‘http://<server_ip>:9393’, create a local username and log with it.

## Configure Master & Slave for DNS replication


## Load zone from Bind
### For master server
Copy all files zone in a directory like /tmp/zones on your PowerDNS server.
Create a file named ‘data’ with the list of all zones you want to import like :
zone1
zone2
zone3
...

On master create a file named ‘load-master-zone.sh’ file it with :
```
#!/bin/bash
FILES=/home/exploit/zones/*
while IFS= read -r file
do
  echo "Processing $file file..."
  # take action on each file. $f store current file name
  sudo pdnsutil load-zone $file $file
  sudo pdnsutil set-kind $file master
  sudo pdnsutil set-meta $file SOA-EDIT-API INCEPTION-INCREMENT
done < data
```

### For slave server
Create a file named ‘data’ with the list of all zones you want to import like :
zone1
zone2
zone3
...

On slave create a file named ‘load-slave-zone.sh’ file it with :
```
#!/bin/bash
FILES=/home/exploit/zones/*
while IFS= read -r file
do
  echo "Processing $file file..."
  # take action on each file. $f store current file name
  sudo pdnsutil create-slave-zone $file 172.22.0.10
done < data
```
## Appendix
ns1

Create a zone and set it as master

sudo pdnsutil create-zone <zone> <nameserver>

sudo pdnsutil set-kind <zone> master

sudo pdnsutil set-meta <zone> SOA-EDIT-API INCEPTION-INCREMENT

ns2

Create a zone and set it as slave

sudo pdnsutil create-zone <zone> <nameserver>

sudo pdnsutil set-kind <zone> slave


