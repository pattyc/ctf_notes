# Linux Forensics

# Task 1: Introduction

- Linux is used in web servers

# Task 2: Linux Forensics

- Many distros

# Task 3: OS and account information

This exercise is all done in ubuntu

```
cat /etc/os-release
cat /etc/passwd
cat /etc/group
sudo cat /etc/sudoers
```

- /var/log/btmp: saves information about failed logins
- /var/log/wtmp: keeps historical data of logins
- need to use the last utility to read them
- /var/log/auth.log 

```
sidequest
we have credentials for the host but I can't ssh due to a public key error

(kali㉿kali)-[~]
└─$ ssh 10.10.233.35 -l ubuntu
ubuntu@10.10.233.35: Permission denied (publickey).

also tried vnc but it is bound to localhost

The lab guide says we have creds for the ubuntu user but password authentication is disabled in the ssh config

Using the split view in tryhackme I had to do the following

so changed
/etc/ssh/sshd_config
PasswordAuthentication yes

additionally the password they gave us was wrong for the ubuntu user, but was able to sudo to root and set it to what they said it would be set to

can now ssh to the host and start the lab
```

## Q1
- In the attached VM, there is a user account named tryhackme. What is the uid of this account?
cat /etc/passwd

A: 1001

## Q2
- Which two users are the members of the group audio?
cat /etc/group

A: ubuntu,pulse

## Q3
- A session was started on this machine on Sat Apr 16 20:10. How long did this session last?
last
reboot   system boot  5.4.0-1029-aws   Sat Apr 16 20:10 - 21:43  (01:32)
A: 01:32

# Task 4: System Configuration

```
/etc/hostname
/etc/timezone
/etc/network/interfaces
ip a
netstat -natp
ps aux
/etc/hosts
/etc/resolv.conf
```
The next batch of questions are pretty cruisy
```
ubuntu@Linux4n6:~$ cat /etc/os-release 
NAME="Ubuntu"
VERSION="20.04.1 LTS (Focal Fossa)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 20.04.1 LTS"
VERSION_ID="20.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=focal
UBUNTU_CODENAME=focal
ubuntu@Linux4n6:~$ cat /etc/hostname
Linux4n6
ubuntu@Linux4n6:~$ cat /etc/timezone
Asia/Karachi                                
ubuntu@Linux4n6:~$ ss -plant
State         Recv-Q        Send-Q               Local Address:Port                Peer Address:Port         Process                                    
LISTEN        0             5                        127.0.0.1:5901                     0.0.0.0:*             users:(("Xtigervnc",pid=917,fd=7))        
LISTEN        0             100                        0.0.0.0:80                       0.0.0.0:*                                                       
LISTEN        0             4096                 127.0.0.53%lo:53                       0.0.0.0:*                                                       
LISTEN        0             128                        0.0.0.0:22                       0.0.0.0:*                                                       
LISTEN        0             5                        127.0.0.1:631                      0.0.0.0:*                                                       
ESTAB         0             80                    10.10.233.35:22                    10.4.17.48:55904                                                   
LISTEN        0             5                            [::1]:5901                        [::]:*             users:(("Xtigervnc",pid=917,fd=8))        
LISTEN        0             128                           [::]:22                          [::]:*                                                       
LISTEN        0             5                            [::1]:631                         [::]:*                                                       
ubuntu@Linux4n6:~$ which Xtigervnc
/usr/bin/Xtigervnc
```
- Read about the flags used above with the netstat and ps commands in their respective man pages.
```
ps on it's own seems to just show all processes attached to the current user?
-a lifts the 'only yourself' restriction
-x lifts the 'must be attacked to a tty' restriction
-u displays euid of process
```

```
ss, when no option is used, displays a list of open non-listening sockets (e.g. TCP/UNIX/UDP) that have established connection.

-p list process name
-l listening only
-a display listenening and non-listening (tcp adds established)
-n don't resolve service names
-t tcp

I just learnt that specifying -l and -a is pointless, will probs start using "ss -pant", which is actually what this guide uses for netstat
```

# Task 5: Persistence mechanisms
```
/etc/crontab
/etc/init.d
~/.bashrc
/etc/bash.bashrc
/etc/profile
```

## Q1
- In the bashrc file, the size of the history file is defined. What is the size of the history file that is set for the user Ubuntu in the attached machine?
A: 2000

# Task 6: Evidence of Execution
Sudo execution history
```
$ cat /var/log/auth.log* |grep -i COMMAND|tail
```

Bash history
- check all users bash_history files
```
cat ~/.bash_history
```

Files accessed using vim
```
cat ~/.viminfo
```

## Q1
- The user tryhackme used apt-get to install a package. What was the command that was issued?
```
sudo -i
cat /home/tryhackme/bash_history
```
A: sudo apt-get install apache2

## Q2
- What was the current working directory when the command to install net-tools was issued?
```
root@Linux4n6:~# cat /home/ubuntu/.bash_history 
ls -a
ls -al
unlink .bash_history
ls -a
rm -rf bash_logout
history -w
ls -a
ls -al
unlink .bash_history
ls -a
rm -rf bash_logout
history -w
ls -a
cat .bash_history 
sudo adduser tryhackme
cat /etc/passwd
last -f /var/log/btmp 
sudo last -f /var/log/btmp 
sudo last -f /var/log/wtmp 
vncpasswd
netstat -natp
cat /etc/passwd
sudo vim /etc/hostname
cat .bash_history
sudo apt-get install net-tools
netstat -natp
sudo last -f /var/log/wtmp
sudo dpkg-reconfigure tzdata 
sudo reboot
```
The user did not change directories so just assumed ubuntu user's home
A: /home/ubuntu

# Task 7: Log files

```
/var/log/syslog*
/var/log/auth.log
third party logs also contained in /var/log
```

## Q1
- Though the machine's current hostname is the one we identified in Task 4. The machine earlier had a different hostname. What was the previous hostname of the machine?

```
grep -ir hostname /var/log
syslog.1:Apr 17 04:40:12 tryhackme NetworkManager[540]: <info>  [1650170412.7585] hostname: hostname changed from (none) to "tryhackme"
```

A: tryhackme