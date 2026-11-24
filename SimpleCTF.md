1. **How many services are running under port 1000?**
	Scanned given machine using NMAP
	```sudo nmap -Pn ip	```
	Ans: 2

2. **What is running on the higher port?**
	Ran NMAP with -sV switch 
	```nmap -Pn -p 2222 -sV ip	```
	Ans: ssh

- Used Gobuster for directory discovery for some extra info
	```gobuster dir -u http://ip/ -w directory-wordlist.txt ```	
	found /simple directory hosting cms made simple site

3. **What's the CVE you're using against the application?**
	Searched for CMS made Simple and found the CVE related to it
	Ans: CVE-2019-90**

4. **To what kind of vulnerability is the application vulnerable?**
	Found on exploit db page- https://www.exploit-db.com/exploits/46635
	Ans: sq**

5. **What's the password?**
	- I used the python script provided on the exploit.db for the vulnerablity and cracked the username and password

	`python exploit.py -u http://ip/simple/ -w /usr/share/wordlist/rockyou.txt`
	
	username: mat**
	password: sec***

6. **Where can you login with the details obtained?**
	Ans: ssh

7. **What's the user flag?**
	- Login ssh using the cracked username and password and then use cat to read the user.txt file to get the flag
	
	`ssh: ssh -p 2222 username@ip`
	
	Ans: G00d j**, k*** up!

8. **Is there any other user in the home directory? What's its name?**
	- ls /home to see
	Ans: sun****

9. **What can you leverage to spawn a privileged shell?**
	- I tried to access sunbath and root directory but i couldn,t. So, i ran `sudo -l` to see what commands i can run as match. I found that match can run `vim` as root and we can try to use to gain root access.
	
	Ans: vim

10. **What's the root flag?**
	- Type: `sudo /usr/bin/vim` or `sudo vim` then type `:!bash` and boom i got the root user
	- go to /root and then cat `root.txt`
	
	Ans: W3ll d0n3. You made it!


Solving this CTF i learned to search for exploits and how to use them to exploit a vulnerability. 