# Using Synology DS916+'s Internal Postgres Server

Using the internal Postgresql v11.11 server on a Synology DS916+ NAS.

As of 2022-07-01, Postgresql version 11.11 is running internally on the Synology DS916+ NAS at its current `DSM 7.1-42661 Update 2` patch level. 

Beware that Postgresql v11.11 is set to expire Long Term Support as of November 9th, 2023 (5 months from the date of this writing). Keep this in mind before investing heavily in this approach. The current Postgresql LTS is version 14.4 and expires in November 2026 (four years from date of this writing). You may prefer to install a fresh postgres docker instead.

Note that these instructions do work, but are incompleted. They are lacking any research into which volume which stores the database, and whether it has sufficient available space for supplementary use, and is not restricted in size or any other way.

Having been warned, here are instructions for getting access to the internal Postgresql v11.11 database server.



### Prerequisites

0. A running NAS DS916+, server having SSH login enabled
0. Non-password login for above, using .ssh/authorized_hosts and RSA keys
0. Postresql client installed on your workstation (i.e. psql)



### Names in use

Throughout this example I am using the following system/usernames. Change the commands as appropriate for your environment. Note that this is relying on creating the same postgresql username/password as the Synology NAS username/password. If this is a security concern you will have to make adjustments accordingly.

| name       | represents                         | 
| ----       | ----------                         |
| `DS16`     | hostname of NAS server             |
| `pbnelson` | username on NAS server             | 
| `SECRET`   | password for above                 |
| `pbndb`    | name of new database being created | 



## Creating new Postgresql database and user

You should be able to use these commands to see the current Synology database. This is your live system, so be careful not to mess up any data. You have been warned.

````bash
~$ ssh DS16

Synology strongly advises you not to run commands as the root user, who has
the highest privileges on the system. Doing so may cause major damages
to the system. Please note that if you choose to proceed, all consequences are
at your own risk.

pbnelson@DS16:~$ sudo psql -U postgres
Password: SECRET
psql (11.11)
Type "help" for help.

postgres=#
postgres=# \l
                                   List of databases
    Name     |   Owner    | Encoding  |  Collate   |   Ctype    |   Access privileges   
-------------+------------+-----------+------------+------------+-----------------------
 autoupdate  | postgres   | SQL_ASCII | C          | C          | 
 mediaserver | MediaIndex | UTF8      | en_US.utf8 | en_US.utf8 | 
 postgres    | postgres   | SQL_ASCII | C          | C          | 
 synoindex   | MediaIndex | SQL_ASCII | C          | C          | 
 template0   | postgres   | SQL_ASCII | C          | C          | =c/postgres          +
             |            |           |            |            | postgres=CTc/postgres
 template1   | postgres   | SQL_ASCII | C          | C          | =c/postgres          +
             |            |           |            |            | postgres=CTc/postgres
(6 rows)

postgres=# 

````

At this point, being connected to the DS916+'s Postgresql server, we will create a new user and database. Be sure to substitute your own username, dbname, password, etc.

````sql
postgres=# -- create a new user (username and password same as shell login, in this case user="pbnelson")
postgres=# create user pbnelson password 'SECRET';
CREATE ROLE
postgres=# -- create new database with name "pbn", takes a few seconds
postgres=# create database pbndb with owner=pbnelson; 
CREATE DATABASE
postgres=# -- connect to new database
postgres=# \c pbndb 
You are now connected to database "pbndb" as user "postgres".
postgres=# -- list tables (there are none, yet)
postgres=# \dt 
postgres=# -- quit postgresql psql client
pbndb=# \q
pbnelson@DS16:~$ 

````



### Testing new Postgresql database and user connectivity

It should now be possible to directly connect to the new Postgresql database, using the new Postgresql username, from the NAS ssh shell.

````sql
pbnelson@DS16:~$ # note that psql defaults to the same d/b username as shell username.
pbnelson@DS16:~$ psql pbndb
psql (11.11)
Type "help" for help.

pbndb=> -- quit psql
pbndb=> \q
pbnelson@DS16:~$ 

````



### Updating Synology SSH and Postgresql configuration for remote connectivity

While still logged into the NAS ssh shell, run the following commands to enable port forwarding and remote database connectivity.

Note that these commands may have to be re-run after a Synology NAS system software update.

````bash

# backup postgresql and ssh config files
pbnelson@DS16:~$ sudo cp /etc/postgresql/pg_hba.conf ~/pg_hba.conf.orig
pbnelson@DS16:~$ sudo cp /etc/ssh/sshd_config ~/sshd_config.orig

# enable port forwarding by changing only the *first* line in sshd_config
# that reads 'AllowTcpForwarding no' to 'AllowTcpForwarding yes'
pbnelson@DS16:~$ sudo sed --in-place '0,/AllowTcpForwarding no/ s//AllowTcpForwarding yes/' /etc/ssh/sshd_config

# enable postgres connections from external network addresses 
# (other than localhost) by adding this line to the end of 
# the pg_hba.conf file:  'host    pbndb           all             127.0.0.1/32            md5'
pbnelson@DS16:~$ echo "
# used for remote login
host    pbndb           all             127.0.0.1/32            md5
" | sudo tee -a /etc/postgresql/pg_hba.conf

# restart NAS to make new SSH and Postgresql configurations take effect
pbnelson@DS16:~$ sudo reboot

````



### Connecting from your local dev workstation

Run these commands on your local dev workstation, *not* on the NAS itself.


#### If you need to install the postgresql client (psql)

On MacOS:
````bash
# install postgresql client (psql) on mac, then link it to the PATH variable
brew install libpq
echo 'export PATH="/opt/homebrew/opt/libpq/bin:$PATH"' >> /Users/peter/.bash_profile
source ~/.bash_profile

````

On Linux or Windows:
````bash
# google it
````


#### Once you have the `psql` client installed


Note that this requires passwordless SSH login to the NAS server. If you don't have that working, google how to use RSA keys `~/.ssh/id_rsa.pub` and `~/.ssh/authorized_keys` to enable that.

````bash
# open a tunnel and connect sample command to connect remotely
~$ port=55443 # pick a random port
~$ ssh -f -L ${port}:DS16:5432 pbnelson@DS16 sleep 1 && \
PGPASSWORD="SECRET" psql --host=127.0.0.1 --port=${port} --user=pbnelson --dbname=pbndb


psql (14.4, server 11.11)
Type "help" for help.

pbndb=> -- list tables
pbndb=> \dt
Did not find any relations.
pbndb=> -- quit
pbndb=> \q
~$ 

````
