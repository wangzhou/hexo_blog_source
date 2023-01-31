---
title: ubuntu使用LDAP做用户管理
tags:
  - 运维
  - LDAP
description: >-
  LDAP is one way to implement centralized authentication in linux system. This
  note talks about how to make related configurations to implement centralized
  authentication in ubuntu 14.04.
abbrlink: 671c9ad8
date: 2021-07-11 23:50:39
categories:
---

The basic view about centralized authentication using LDAP are as below:
1. We need a server(A) to install LDAP server which has the accounts'
   information and deals with authentication requests.
2. We need to install LDAP client in another server(B) which you want to login.
3. You can login server B after adding relative account in LDAP server. Or if
   you already have some accounts information, you can login from another
   server using LDAP server.

Configure LDAP server
---------------------
1. sudo apt-get install slapd ldap-utils
   During the install process, it needs to fill in some basic configurations.
   You can also use command below to change them.

2. sudo dpkg-reconfigure slapd
   Refer to [1] to see how to configure.

3. Add groups, users.
   You can use ldif file to do this, another way is to use ldapscripts.
   (1) ldif way
       a. Edit ldif file as below: vi add_content.ldif
       ...
       b. Add users, groups, users' directory and groups' directory as below;
          ldapadd -x -D cn=admin, dc=company, dc=com -W -f add_content.ldif

       it will appear:

       Enter LDAP Password:
       adding new entry "ou=User,dc=company,dc=com"
       ...

       NOTE: if failed there, mostly you should check basic slapd configure.

       c. ldapdelete -x -D "cn=admin,dc=company,dc=com" -W "uid=test,ou=User,dc=company"
          Using above command to delete one entry.

	  ldapsearch -x -LLL -b dc=company,dc=com
          Using above command to search information of all entries.

   (2) ldapscripts way
       a. sudo apt-get install ldapscripts

       b. Configure /etc/ldapscripts/ldapscripts.conf:
          SERVER=localhost
	  SUFFIX="dc=company,dc=com"
	  GSUFFIX="ou=Groups"
	  USUFFIX="ou=Users"
	  BINDDN="cn=admin,dc=company,dc=com"
          BINDPWDFILE="/etc/ldapscripts/ldapscripts.passwd"

       c. echo -n "your_root_passwd_for_ldap" > /etc/ldapscripts/ldapscripts.passwd

       d. User relative command to add group, user: ldapadduser, ldapaddgroup

       NOTE: We can add an account by ldapadduser account_name group_name.
             e.g. ldapadduser test User
	     Use c step above to write password to ldapscripts.passwd. This
	     will *NOT* write "\n" at the end of password line.

Configure LDAP client[2]
------------------------
1. sudo apt-get install libpam-ldap nscd
   When installing libpam-ldap, it will ask you to configure LDAP client during
   the install process. You can also change the configuration as below.

2. sudo apt-get install ldap-auth-config
   sudo dpkg-reconfigure ldap-auth-config[2]

3. Modify /etc/nsswitch.conf to choose how to make authentication:
       passwd: files ldap
       group: files ldap
       shadow: files ldap

   NOTE: You'd better put "filles" befort "ldap"

4. Build home directory automatically in LDAP client[3]
   add line: session required pam_mkhomedir.so skel=/etc/skel/ umask=0022
   to /etc/pam.d/common-account

Migrate accounts information
----------------------------
If you already have users information in /etc/passwd, /etc/group, /etc/shadow,
and you want to use LDAP manage users information, you can do as below:

1. sudo apt-get install migrationtools

2. Modify /etc/migrationtools/migrate_common.ph:
      $DEFAULT_BASE = "dc=company,dc=com";

3. /usr/share/migrationtools/migrate_passwd.pl /etc/passwd add_people.ldif

4. Modify add_people.ldif:
   Change the information about group for every user.

   Copy the encrypted pass word in /etc/shadow to replace "x" in "userPassword: {crypt}x"
   attribution in add_people.ldif

   FIXME...

5. ldapadd -x -D cn=admin,dc=company,dc=com -W -f add_people.ldif

NOTE: I just need the password information here. If you want login using LDAP,
      you should also consider to migrate group information in /etc/group

Backup existed LDAP date
------------------------
...

Reference:
- https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-a-basic-ldap-server-on-an-ubuntu-12-04-vps
- https://www.digitalocean.com/community/tutorials/how-to-authenticate-client-computers-using-ldap-on-an-ubuntu-12-04-vps
- http://www.debian-administration.org/article/403/Giving_users_a_home_directory_automatically
