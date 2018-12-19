External ACL for Squid (external MSSQL database)
======================

External ACL to use on public computers.

How it works
------------

```ext_sql_browzine_acl``` is written in perl and starts from ```/etc/squid.conf```. Make sure to set ```children-max``` to a proper value to reflect number of users. If 200 users a fork of 20 concurrent processes will be a good start. This depend on how fast the database will answer your request. Slow response time, increase the number. A "starndard" linux RHEL server 8 RAM will easily work with 60-100 cuncurrent processes without any problem.

Make sure to use this ACL late in the squid.conf to avoid unnecessary lookups to database. 


How to use
----------

Make sure you have a database with all URIs provided from "Third Iron" (support for Browzine). Populate database with two columns URI and DATE. Add the extarnal ACL ```ext_sql_browzine_acl ``` as a lookup in squid.conf file. 

Tests
-----
To test the ACL before production and when changes, run this in shell:

```
./ext_sql_browzine_acl --debug --user "dbuser_name" --password "dbuser_password" --server "ip or fqdn" --table "dbo.br" --database "browzine"
```

Write some URLs and look for the terminal output:
* http://urlproxy.sunet.se
* urlproxy.sunet.se
* https://urlproxy.sunet.se/canit/urlproxy.php?_q=aH
* sr.se (not in database) ERR

Debug
-----

Usage of ``` --debug``` flag will print lines to STDERR in this case handled by Squid ```/var/log/squid/cache.log```

### Example

When launch ```ext_sql_browzine_acl``` with debug on it'll print connection string to STDERR one time / fork:
```DBG - DBI:ODBC:Driver={ODBC Driver 17 for SQL Server};Server=ip;Database=browzine;Uid=dbuser_name;Pwd=dbuser_password```

#### ext_sql_browzine_acl
Send ```muse.jhu.edu``` to ```ext_sql_browzine_acl``` with debug on:

```
muse.jhu.edu
DBG - String received: muse.jhu.edu
DBG - Parsed string to serach for: muse.jhu.edu
DBG - SELECT * FROM browzine.dbo.br WHERE URL LIKE %muse.jhu.edu%
DBG 0 - url:  http://muse.jhu.edu   date: 2018-12-30 00:00:00.000
DBG 1 - url:  http://muse.jhu.edu/  date: 2018-12-30 00:00:00.000
DBG 2 - url:  https://muse.jhu.edu/  date: 2018-12-30 00:00:00.000
DBG - DB connection is closed!
OK
```

Send ```https://urlproxy.sunet.se/canit/urlproxy.php?_q=aHR0cDovL25ldXJvaW50ZXJ2ZW50aW9uLm9yZy8``` to ```ext_sql_browzine_acl``` with debug on:

```
DBG - String received: https://urlproxy.sunet.se/canit/urlproxy.php?_q=aHR0cDovL25ldXJvaW50ZXJ2ZW50aW9uLm9yZy8
DBG - Parsed string to serach for: urlproxy.sunet.se
DBG - SELECT * FROM browzine.dbo.br WHERE URL LIKE %urlproxy.sunet.se%
DBG 0 - url:  https://urlproxy.sunet.se/canit/urlproxy.php?_q=aHR0cDovL2FtaGNham91cm5hbC5vcmcv&_s=ZnJlZHJpay5seXNlbkB1Yi51dS5zZQ%3D%3D&_c=2636fed4&_r=dXUtc2U%3D  date: 1899-12-30 00:00:00.000
DBG 1 - url:  https://urlproxy.sunet.se/canit/urlproxy.php?_q=aHR0cDovL2NqY21oLmNvbS8%3D&_s=ZnJlZHJpay5seXNlbkB1Yi51dS5zZQ%3D%3D&_c=35dad620&_r=dXUtc2U%3D  date: 1899-12-30 00:00:00.000
...
...
DBG 51 - url:  https://urlproxy.sunet.se/canit/urlproxy.php?_q=aHR0cDovL3d3dy50b3hpY29sb2d5aW50ZXJuYXRpb25hbC5jb20v&_s=ZnJlZHJpay5seXNlbkB1Yi51dS5zZQ%3D%3D&_c=daed197e&_r=dXUtc2U%3D  date: 1899-12-30 00:00:00.000
DBG 52 - url:  https://urlproxy.sunet.se/canit/urlproxy.php?_q=aHR0cDovL3d3dy50cmFuc3Jlc3BtZWQuY29tLw%3D%3D&_s=ZnJlZHJpay5seXNlbkB1Yi51dS5zZQ%3D%3D&_c=db117adc&_r=dXUtc2U%3D  date: 1899-12-30 00:00:00.000
DBG - DB connection is closed!
OK
```

Send ```http://aftonbladet.se``` to ```ext_sql_browzine_acl``` with debug on:

```
DBG - String received: http://aftonbladet.se
DBG - Parsed string to serach for: aftonbladet.se
DBG - SELECT * FROM browzine.dbo.br WHERE URL LIKE %aftonbladet.se%
DBG - Could not find http://aftonbladet.se in database
DBG - DB connection is closed!
ERR

```

Installation RHEL7
------------------
##### Use squid repo

```
rpm -i http://ngtech.co.il/repo/centos/7/squid-repo-1-1.el7.centos.noarch.rpm
yum install squid-helpers
systemctl start squid
systemctl enable squid
```

##### Install ODBC for MSSQL (will also install unixODBC)

```
curl https://packages.microsoft.com/config/rhel/7/prod.repo > /etc/yum.repos.d/mssql-release.repo
yum install msodbcsql17
```

##### ODBC for DBI perl

```
yum install perl-DBD-ODBC
vim /etc/odbcinst.ini
```

##### Firewalld

If usage of firewalld make sure its open for both squid and database

Nice to know
------------
Some installation scratchpads.

### MariaDB MySQL

```
DROP TABLE IF EXISTS `sessions`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `sessions` (
  `url` varchar(256) NOT NULL,
  `date` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`url`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
/*!40101 SET character_set_client = @saved_cs_client */;
```

### PostgresSQL

```
DROP DATABASE browzine;

CREATE DATABASE browzine WITH TEMPLATE = template0 ENCODING = 'UTF8' LC_COLLATE = 'en_US.UTF-8' LC_CTYPE = 'en_US.UTF-8';

ALTER DATABASE browzine OWNER TO broread;

CREATE TABLE public.sessions (
    url character varying(256) NOT NULL,
    enabled timestamp without time zone DEFAULT CURRENT_TIMESTAMP NOT NULL
);


ALTER TABLE public.sessions OWNER TO broread;


INSERT INTO public.sessions (url, enabled) VALUES ('http://muse.jhu.edu/', '2018-12-18 13:33:48.76027');
INSERT INTO public.sessions (url, enabled) VALUES ('http://nr.ucpress.edu/', '2018-12-18 13:33:58.795883');
INSERT INTO public.sessions (url, enabled) VALUES ('https://urlproxy.sunet.se/canit/urlproxy.php?_q=aHR0cDovL25ldXJvaW50ZXJ2ZW50aW9uLm9yZy8%3D&_s=ZnJlZHS5zZQ%3D%3D&_c=7c4f6009&_r=dXUtc2U%3D', '2018-12-18 13:34:15.298673');
INSERT INTO public.sessions (url, enabled) VALUES ('https://urlproxy.sunet.se/canit/urlproxy.php?_q=aHR0cDovL21vc3F1aXRvLWphbWNhLm9yZy8%3D&_s=ZnJlZHJpayZQ%3D%3D&_c=3c573f25&_r=dXUtc2U%3D', '2018-12-18 13:34:34.462326');
INSERT INTO public.sessions (url, enabled) VALUES ('http://mmbr.asm.org/', '2018-12-18 13:34:45.702683');
INSERT INTO public.sessions (url, enabled) VALUES ('https://agricultureandfoodsecurity.biomedcentral.com/', '2018-12-18 13:35:16.180128');



ALTER TABLE ONLY public.sessions
    ADD CONSTRAINT sessions_pkey PRIMARY KEY (url);

GRANT ALL ON SCHEMA public TO PUBLIC;
```

TODO
----

* Some comments in code.
* Do URI column i dabase primary key to avoid duplicates.
* Use API to populate database with new URIs
