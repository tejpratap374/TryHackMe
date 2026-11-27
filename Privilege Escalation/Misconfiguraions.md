## Categories Of Misconfiguration
**Common Misconfigurations**

1. User and Group Configuration
	- Assigning user accounts to administrative group unnecessarily
	- Creating Service accounts with excessive privilege
	- Failing to disable or remove default accounts that ship with OS
	- Configuring weak or absent password policies that allows easily guessable credentials

 2. File and Directory Permissions

	- **Linux:**
	    
	    - **Unnecessary SUID binaries:** The SUID (Set User ID) bit makes an executable run with the file owner's privileges rather than the executing user's—escalating privileges if owned by `root` and capable of arbitrary command execution or file access.
	        
	    - **World-writable executables:** Scripts or binaries writable by anyone that are executed by privileged processes.
	        
	    - **Overly broad read permissions:** Sensitive files (such as `/etc/shadow` or SSH private keys) readable by unauthorized accounts.
	        
	- **Windows:**
	    
	    - **Excessive ACLs:** Access Control Lists granting overly broad permissions.
	        
	    - **Writable `PATH` directories:** System `PATH` folders modifiable by non-privileged users.
	        
	    - **Weak installation folders:** Program installation directories with permissive default ACLs.
	        
	    - **Misconfigured registry ACLs:** Registry keys editable by non-privileged users.

3. Service Configurations
	- **Privileged service execution:** Services often run unnecessarily as `root` (Linux) or `LocalSystem` (Windows), giving an attacker elevated access if compromised.
    
	- **Writable configurations or binaries:** If non-privileged users can write to service configuration files or executables, they can alter behavior or replace binaries to escalate privileges.
	    
	- **Unquoted paths with spaces:** On Windows, if a service executable path contains spaces and lacks quotation marks, the Service Control Manager tests paths step-by-step across spaces.
	    
	- **Path resolution order example:** For `C:\Program Files\My App\service.exe`, Windows tries:
	    
	    1. `C:\Program.exe`
	        
	    2. `C:\Program Files\My.exe`
	        
	    3. `C:\Program Files\My App\service.exe`
	        
	- **Escalation impact:** If a non-privileged user can write to any intermediate folder along that resolution path, dropping a malicious binary at a tested boundary lets it execute with the service account's privileges.
	
4. Schedule Tasks and Cron jobs
	- **Root cause:** Utilities invoked by scheduled tasks with wildcards (`*`) can misinterpret specially crafted filenames as command-line flags.
    
	- **Attack setup:** If a privileged task (such as a root `cron` job running `tar cf /backup/archive.tar *`) operates in a directory where a low-privileged user can create files, the user can plant flag-named files.
	    
	- **Exploit example (`tar`):** Creating files named `--checkpoint=1` and `--checkpoint-action=exec=sh shell.sh` causes `tar` to parse them as options when expanding `*`.
	    
	- **Privilege escalation:** `tar` executes `shell.sh` directly with the elevated privileges of the scheduled task (e.g., `root`).