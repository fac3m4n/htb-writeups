# HackTheBox Machine: Mananger - Walkthrough

- HTB box name: Mananger
- OS: Windows
- Difficulty: Medium
- IP: 10.10.11.236

I will be using [Hacktricks](https://book.hacktricks.xyz/welcome/readme) as guide in my walkthrough. It is awesome resource and includes lots of useful information when pentesting.

## Scanning

Lets scan againg for known ports with some extra flags:

- sC : Default script scan
- sV : Service version detection
- oA : Output to file(to keep the record)

![nmapResult](/images/image-15.png)

There are a lot of open ports, lets list the interesting ones to tackle with:

1. 80(HTTP) - Web server
2. 445(SMB) - File sharing
3. 1433(MSSQL) - SQL server
4. 3268(LDAP) - LDAP
5. 3268(LDAP) - LDAP over SSL

## 80(HTTP) - Web server

LDAP ports revealed Domain Names, lets add them to /etc/hosts file

`echo 10.10.11.236  dc01.manager.htb mananger.htb >> /etc/hosts`

Upon checking the website couldn't find much in client side source code.
Was simple website with contact form. And form action was empty so it does not preform any server side actions.

![formNoAction](/images/image-16.png)

Lets preform directory bruteforcing with gobuster and see if we can find something.

`gobuster dir -u 10.10.11.236 -w /usr/share/dirb/wordlists/common.txt`

![gobuster](/images/image-1.png)

There was not much of interesting directories.
Lets move forward.

## 445(SMB) - File sharing

This is interesting one. Most of times SMB is accesible as anonymous user.
Will be using [crackmapexec](https://github.com/byt3bl33d3r/CrackMapExec), which is powerfull tool to enumerate Windows/Active Directory environments, to try to access SMB and bruteforce AD objects.

`crackmapexec smb 10.10.11.236 -u anonymous -p "" --rid-brute 10000`

![smbScan](/images/image.png)

We were able to get usernames. Lets tidy them up and save them in `users.txt` file so that we can use them in bruteforcing. Running bruteforcing with huge list can take a long time. We can try to go faster route by trying usernames as passwords. Human nature is lazy to use complicated passwords sometimes :D.
We can again use `crackmapexec`, by specifying user and password list with additinal flags.
`crackmapexec smb 10.10.11.236 -u users.txt -p users.txt --no-brute --continue-on-success`

![smbScan2](/images/image-2.png)

It worked for `operator` user. Lets look into shares and see if we can find something.

Will be using `smbclient` to list the shares

![smbshares](/images/image-17.png)

So we have common share names for Windows machines as expected.

Tried to list files in `C$` share but it did not work.

![smbCaccessDenied](/images/image-18.png)

## 1433(MSSQL) - SQL server

Lets try `operator` user to enumarate MSSQL server as well. We will be using same `crackmapexec` command as above but for mssql.
`crackmapexec mssql 10.10.11.236 -u operator -p operator --no-brute --continue-on-success`

![mssql](/images/image-3.png)

We can see that operator can login into MSSQL server. Let login using mssql client and try to find some usefull data. Will be using mssql client from impacket.
`impacket-mssqlclient -port 1433 10.10.11.236/operator:operator@10.10.11.236 -window`

![mssql1](/images/image-4.png)

Lets look around and see if we can find something. Help command is handy here and we can run the following commands:

![impacketHelp](/images/image-5.png)

I have enumerated db, no interesting info there.
If we want to have reverse shell we need to access to cmd `xp_cmdshell`.
We cna check for permissions if we can use that command `EXEC sp_helprotect '$commandName'`

![checkPermissions](/images/image-19.png)

So we do not have permissions to use `xp_cmdshell` command.
But we have permission for `xp_dirtree` to list the directories. And try to look for interesting files.

![dirtree](/images/image-7.png)

We can see directories and lets try to go into `Users` directory, and see if we have something there.
We can see potential user Raven, but the folder is empty

![userFolder](/images/image-20.png)

Simple [google search](https://serverfault.com/questions/281159/finding-the-root-for-a-windows-iis-server) will tell us `c:\Inetpub\wwwroot` is default IIS directory. We can take a look there.

![iisDirectory](/images/image-8.png)

We found `website-backup-27-07-23-old.zip` file and we can download it with wget and see whats in there.

![wget](/images/image-9.png)

Found `.old-conf.xml` file. And there were credentials for previous user that we found: `raven`

![oldconf](/images/image-10.png)
![userData](/images/image-11.png)

We can test if that credentials are still correct with `crackmapexec`.
`crackmapexec winrm dc01.manager.htb  -u raven -p "R4v3nBe5tD3veloP3r!123" -d manager.htb --no-brute --continue-on-success`

![crackWinrm](/images/image-12.png)

Appears that we can access using these credentials. We will utilize this credentials to get acces into system shell using [evil-winrm](https://github.com/Hackplayers/evil-winrm).
`evil-winrm -i 10.10.11.236 -u raven -p 'R4v3nBe5tD3veloP3r!123'`

![evilwinrm1](/images/image-13.png)

I found `Certify.exe` which is used for privelage escalation in Windows machines, especially for  Active Directory certificate abusing. This is can be a hint for us to escalate privelages.

Before digging into it, lets see what we can find user flag in users Desktop
![userflag](/images/image-14.png)

Here is our user flag.

## Privelage Escalation

As this is machine is active directory I wanted to follow [Active Directory Methodology](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology) methods to escalate privelages.
First we need to check what privelages does user Raven have.
`whoami /priv` command can help us with that.

![userPriv](/images/image-21.png)

We can see that user Raven has `SeMachineAccountPrivilege` privelages that stand out.
I will try to run [winpeass.exe](https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS) to see if we can find something interesting.
We can upload it using `upload` command.

![winpeas](/images/image-22.png)

It gave huge list of possible things we can do in order to attack.

And also I made google search for `SeMachineAccountPrivilege` abuses and found [WindowsAD Escalation](https://github.com/0xJs/RedTeaming_CheatSheet/blob/main/windows-ad/Domain-Privilege-Escalation.md) article.

I was not able to make privelage escalation for now. Will need some time digging into it. I will be back with pwned machine soon. Stay tuned.
