CCTV HTB Walkthrough
This walkthrough covers the CCTV machine from Hack The Box.
After adding cctv.htb to the /etc/hosts file, we can start attacking the machine.

STEP 1: Network Enumeration
First, scan the target IP address to identify open ports and discover possible attack surfaces.
In this machine, only two ports are open:
22 → SSH
80 → Web application



STEP 2: Web Application Discovery
Now we start enumerating the web application.
Tools like:

gobuster

can be useful here.
After some crawling, you will discover a login page under:
http://cctv.htb/zm
If you are lucky, trying default admin credentials may work.

Admin Page Enumeration
After exploring the admin panel, we can gather some useful information for our attack plan.
The most important finding is:
ZoneMinder v1.37.63

What is ZoneMinder?
ZoneMinder is a free, open-source, and highly scalable video surveillance system used for monitoring CCTV and IP security cameras.

ZoneMinder SQL Injection (CVE-2024-51482)
After researching the installed version, we discover that it is vulnerable to a Boolean/time-based blind SQL injection vulnerability.
The most useful tool for exploiting this vulnerability is:

sqlmap

Unlike normal SQL injection vulnerabilities, this one does not return query results directly in the HTTP response.
Instead, it is exploitable through time-based blind SQL injection, where information is inferred from server response delays.
For more information:

https://github.com/ZoneMinder/zoneminder/security/advisories/GHSA-qm8h-3xvf-m7j3


STEP 3: Database Enumeration and Hash Cracking
Using sqlmap, dump credentials from the Users table in the zm database.
You will find:

usernames
hashed passwords

Save the password hashes and crack them offline using tools such as: john

STEP 4: Initial Access via SSH
After cracking the password for user mark, use the credentials to access the machine through SSH.
Once inside:

search for user.txt
inspect important files
check for privilege escalation vectors
enumerate sudo permissions and SUID binaries

WHATTT?!!
Nothing useful.
So now we need deeper enumeration.
Interesting locations to inspect:

/tmp 
/opt
/etc/systemd/system

At this stage, focus on:

configuration files
credentials
internal services

Since the machine is related to CCTV infrastructure, it makes sense to investigate monitoring-related services.
One particularly interesting file is:

/etc/systemd/system/motioneye.service

Checking the service configuration reveals another useful file:

/etc/motioneye/motion.conf

Inside the configuration, we discover:

the service runs as root
it is accessible only locally
the application version is 0.43.1b4
And most importantly:
the configuration file contains admin credentials.


STEP 5: Accessing the motionEye Web Interface
Since the service is only accessible locally, we can use SSH port forwarding.
Command:

ssh -L 8765:127.0.0.1:8765 mark@cctv.htb

Now access the application in the browser:

http://127.0.0.1:8765

Login using the credentials found in the configuration file.

STEP 6: Privilege Escalation — motionEye Command Injection (CVE-2025-60787)
After researching vulnerabilities affecting motionEye v0.43.1b4, we discover a command injection vulnerability.
The vulnerable field allows command execution as root.
However, the application performs client-side validation before accepting the payload.

This validation can be bypassed by opening the browser console (F12) and executing:
configUiValid = function() { return true; };

After bypassing the validation, inject your payload to execute commands as root.
For more information:
https://github.com/advisories/GHSA-j945-qm58-4gjx

STEP 7: Obtaining a Root Shell
After sending the payload and setting up a listener, you will receive a root shell on the machine.
Now you have full access.
Flags:
user.txt -> /home/sa_mark/user.txtroot.txt -> /root/root.txt



