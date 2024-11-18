
## Plan

My plan, which is the same for every CTF, will be to enumerate the system using nmap, then procced to gather information on open ports, or services running such as known exploits which i can use to grab onto the system then use privilege escalation and reverse shell techniques to climb the permission ladder and gain full access to the system

## Starting Info

`target ip = 10.10.127.201`

## Enumeration

#### Nmap
nmap is a network mapping tool which can be used to scan systems and perform intellegence gathering enumeration on a system, it is a basic tool for any Cyber Security Agent and will be the starting point of many CTF's. It is a free to use tool that is easy to download and very straight forward to user, in this instance i will be using it in a CLI enviroment on Kali Linux within a virtual machine powered by VMware

i will now scan the target IP using a -sV flag in order to determine services running on the target machine
`nmap -sV 10.10.127.201`

![[Pasted image 20241116172428.png]]

as we can see, there are a few interesting ports that we can look into:
22 - ssh -  a common port used for host secure host communication over ssh
80 - http - this port indicates that there a website we can scan, we will do this next to discover hidden directories
110,143 - pop3,imap - we can see here that this is most likely an email server, meaning if we can gain access we will have lots of exploitable information

deeper level scans returned no further useful infomation

#### Dirb
Dirb or Directory Buster is a web application analysis tool which combined with wordlists from around the internet, can become the most powerful tool in the enumeration phase, as it can discover any page connected to a site, such as : Admin Login Pages, Databases and many more "should be private" pages that a poor web dev would leave open just for us to discover

i will now scan the target, notating my wordlist big.txt and using a -X flag along with .txt to specify that i only want directories and files ending in .txt

`dirb http://10.10.127.201/ /usr/share/dirb/wordlists/big.txt -X .txt`

![[Pasted image 20241116174912.png]]

we will now move onto exploring what we have discovered in the initial enumeration of the machine.
## Info Gathering
After enumeration the machine with appropriate tools, we will now explore our options and delve into everything we have discovered, i plan to access the website, attempt to connect directly to the email ports, and access files location using dirb and follow any leads from there.

#### Website
The Website `10.10.127.201:80`
![[Pasted image 20241116175305.png]]

starting with the website we can see already on the front page they have been subjected to an employee data leak, i will note this down for later. We can also see that staff have been instructed to change passwords, however due to the nature of people, this could be a very strong lead. The attackers have also gained access to the company twitter account, lets explore this


#### Hijacked Twitter
The Company Twitter Account:
![[Pasted image 20241116175851.png]]
Here we can see a pastebin link, pastebin being a free online service where you can upload basic documents anonymously in order to share with others, in this case it has been used to share employee login details. (The actual pastebin link has been taken down due to miscommunication and pastebin believing it was a true data leak, however i have obtained a copy through the magic of the internet)

#### Pastebin
PasteBin Contents:
![[Pasted image 20241116180215.png]]
here we can see a list of usernames, and hashed passwords, i will be using an online hashcracker as it is much easier and quicker to use. This also demonstrates the lack of indepth knowledege required of the nature of hash's showing that this can be done by a true begginer

#### Hashes.Com
Results from Hashes.com:
![[Pasted image 20241116180456.png]]
We now have a list of Unhashed password that can now be used to access a system
(Note the passhash for user stone was not discovered)

Now that we have sufficient infomation to attempt to access the target system we will move to the Execution Phase
## Execution

#### Hydra
We will get straight into it, by using hydra, a multi-threaded bruteforce execution software, it is very straightforward to user, with potentially a very basic knowledge of ports and servers required (such as that of a year 10 computer science lesson)

`hydra -L /home/kali/Documents/users.txt -P /home/kali/Documents/passwords.txt -f 10.10.127.201 pop3`
A few things to break down here, so -L indicating location of the users file, which happens to also be called users in this case. Then -P to state the location of the password file, also named accordingly. the -f flag is used to tell hydra to stop once a successful pair has been determined, followed by the target ip and service we want to access which is pop3 as we discovered that was open in our enumeration

![[Pasted image 20241116182523.png]]

Here we can see that a successful pair has been found. Hurray ! Not Hurray for seinna though, who has failed to follow instructions and because of her negligence her company will now pay the price

#### Telnet
We can now logon to the FowSniff mail server using Telnet, a simple tool to provide a connection between hosts

`telnet 10.10.127.201 110`

target ip, followed by requested port 110, that being the pop3 port discovered earlier
![[Pasted image 20241116183825.png]]
using `user <user>` and `pass <password>` with the aformentioned seina's details we gain acceess to the FowSniff Mail Server

![[Pasted image 20241116184048.png]]
Now using `list` we print the contents of our current working directory, which shows 2 emails, we use `retr` to retrieve the first email

![[Pasted image 20241116184215.png]]
we can see Stone, who we can assume to be the SysAdmin due to the nature of the email, but for now we can see a temporary password given out to users in reaction to the data leak. More importantly we gain the knowledge of a temporary server that we can gain access to. Let's the second email

![[Pasted image 20241116184613.png]]
The key part of this comedic email is the fact that the user baksteen has not read stones email yet, meaning he still has the temporary password in use on his account. Let's see what we can do with this

![[Pasted image 20241116185159.png]]
`ssh baksteen@10.10.127.201 -p22`

so now we are using port 22 over ssh to connect to the FowSniff Server, logged in as baksteen

let's get some permissions !
## Privilege Escalation

![[Pasted image 20241118092233.png]]
by running `groups` we find out what groups baksteen is a part of, we can then find out what files baksteen can edit as being a part of the `users` group, doing this we find a file called `cube.sh`

![[Pasted image 20241118092617.png]]
so now we know we have access to a startup file since we saw this graphic when we initially connected, hinting that this could be our root to the flag

![[Pasted image 20241118093058.png]]
having a deeper look into what files we can access reveals that cube.sh is run on startup, with root privileges, meaning if we can insert a reverse shell script here it will be run and we can get full access to the machine

![[Pasted image 20241118094016.png]]
we now use nano to edit the `cube.sh` file and add a reverse shell pointing to our ip on port 4444 (highlighted orange), all we need to do now is set up a listener on the attacker machine and re-login as baksteen then we should have root privileges

![[Pasted image 20241118094239.png]]
we use `nc` or netcat to set up a listener on the same port we pointed our reverse shell to, ready to recieve a signal and take control

![[Pasted image 20241118094628.png]]
after re-connecting to baksteen's account, the reverse shell is activated, and we use `ls` to list the contents of our directory, with root being the obvious choice to start looking for a flag
![[Pasted image 20241118094821.png]]
low and behold, `flag.txt`

![[Pasted image 20241118094846.png]]
we then get to see the brilliant actual flag to show our work is complete


