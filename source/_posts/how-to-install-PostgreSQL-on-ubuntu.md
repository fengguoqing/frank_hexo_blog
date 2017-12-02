---
title: how to install PostgreSQL on ubuntu
categories: 后端
tags:
  - 教程
  - PostgreSQL
date: 2013-01-13 00:29:56
---

1. This installs the database server/client, some extra utility scripts and the pgAdmin GUI application for working with the database. 
``` shell
$ sudo apt-get install postgresql postgresql-client postgresql-contrib
$ sudo apt-get install pgadmin3 
```
2. Now we need to reset the password for the ‘postgres’ admin account for the server, substitute in the password you want to use for your administrator account. 
``` shell
  $ sudo su postgres -c psql template1 
  template1=# ALTER USER postgres WITH PASSWORD 'password'; 
  template1=# \q 
```
3. That alters the password for within the database, now we need to do the same for the unix user ‘postgres’. 
``` shell
  $ sudo passwd -d postgres 
  $ sudo su postgres -c passwd 
```
4. Set-up the PostgreSQL admin pack that enables better logging and monitoring within pgAdmin. 
``` shell
  $ sudo su postgres -c psql < /usr/share/postgresql/9.1/extension/adminpack--1.0.sql 
```
5. Edit the postgresql.conf file. 
``` shell
  $ sudo gedit /etc/postgresql/9.1/main/postgresql.conf 
```
 Change the line: 
  `#listen_addresses = 'localhost'`
 to 
  `listen_addresses = '*'`
 and also change the line: 
  `#password_encryption = on`
 to 
  `password_encryption = on`

6. Define who can access the server. This is all done using the pg_hba.conf. 
``` shell
  $ sudo gedit /etc/postgresql/8.3/main/pg_hba.conf 
```
 add this text to the bottom of the file: 
  host all all [ip address] [subnet mask] md5 
 add in your subnet mask (i.e. 255.255.255.0) and the IP address of your server (i.e. 138.250.192.115). 
 Note:if you have some password error, please change all "md5" to "trust" in this file. 

7. Now all you have to do is restart the server. 
``` shell
  $ sudo /etc/init.d/postgresql restart 
```
 Reference: 
<http://hocuspokus.net/2008/05/install-postgresql-on-ubuntu-804/>