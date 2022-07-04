# SYNOLOGY_DS916_POSTGRESQL
Using the internal Postgresql server on a Synology DS916+ NAS


## DRAFT! 

These instructions work, but lack context for the SSH tunnel (rskeys, etc). 

Also, further work is needed to ensure that the volume which stores the d/b has sufficient available space and is not restricted in size.



# login to NAS
ssh DS16

# connect to postgresql as "root/admin"
sudo psql -U postgres

# list current databases
\l

# create a new user (username and password same as shell login, in this case user="pbnelson")
create user pbnelson password '<same as login pw>';

# create new database with name "pbn"
create database pbndb with owner=pbnelson;

# connect to new database
\c pbndb

# list tables (there are none, yet)
\dt

# quit psql
\q

# to reconnect to psql from DS16 terminal 
psql pbndb

# quit, again
\q

# backup postgresql and ssh config files
cd ~; sudo cp /etc/postgresql/postgresql.conf ./postgresql.conf.orig
cd ~; sudo cp /etc/postgresql/pg_hba.conf ./pg_hba.conf.orig
cd ~; sudo cp /etc/ssh/sshd_config ./sshd_config.orig

# enable postgres connections from external network addresses (other than localhost)
sudo vi /etc/postgresql/postgresql.conf
DO NOT DO!!! # change line that reads `listen_addresses = '127.0.0.1'` to `listen_addresses = '0.0.0.0'`

# enable port forwarding
sudo vi /etc/ssh/sshd_config
# change line that reads `AllowTcpForwarding no` to `AllowTcpForwarding yes`

# add user to postgres authorized users file
sudo vi /etc/postgresql/pg_hba.conf
# add the following line: `host    pbndb           all             127.0.0.1/32            md5`
# note that you *must* specify `host`, not `local` or the DS16 will reset the entire file to system defaults.

# restart NAS
sudo reboot

# install postgresql client (psql) on mac, then link it to the PATH variable
brew install libpq
echo 'export PATH="/opt/homebrew/opt/libpq/bin:$PATH"' >> /Users/peter/.bash_profile
source ~/.bash_profile

# connect to psql d/b
psql -h DS16 pbndb

# sample command to connect remotely
port=55443 # random
ssh -f -L ${port}:DS16:5432 pbnelson@DS16 sleep 1 && PGPASSWORD="<same-as-login-pw-from-above>" psql --host=127.0.0.1 --port=${port} --user=pbnelson --dbname=pbndb

# quit
\q
