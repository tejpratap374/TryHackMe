## Phase 1 - Situational Awareness

- **Establish host and account context:** Identify current user privileges, group memberships, OS version/architecture, hostname, network role (workstation, web/database server, or domain controller), and domain-joined status before enumerating misconfigurations.
    
- **Guide subsequent checks:** Context shapes what to look for next (for instance, a domain-joined Windows server running IIS differs from a standalone Linux workstation), preventing wasted effort.
    

## Phase 2 - Category-Based Enumeration

- **User and group configuration:** Enumerate users, group memberships, admin/root accounts, default accounts that should be disabled, and accessible password policies.
    
- **File and directory permissions:** Check for overly permissive access—on Linux, SUID/SGID binaries, world-writable files/directories, and broadly readable sensitive files; on Windows, ACLs on system `PATH` folders, installation directories, and service registry keys.
    
- **Service configurations:** List running services and execution accounts, check if binaries or config files are writable by the current user, and inspect unquoted service paths and service security descriptors on Windows.
    
- **Scheduled tasks and cron jobs:** Identify privileged scheduled jobs and check permissions on referenced scripts or binaries via system/user crontabs (Linux) or Task Scheduler (Windows).
    
- **Credential storage:** Look for exposed credentials in shell history, environment variables, config files, registry entries, credential stores, and deployment files—often a direct escalation path.
    
- **Network configuration:** Review listening ports, bound interfaces, and firewall rules to spot unnecessarily exposed services (more tied to lateral movement than local escalation, while clarifying overall posture).
    

## Phase 3 - Prioritisation and Exploitation

- **Rank by directness and impact:** Prioritize findings that offer the most direct, reliable path to high privileges (such as stored admin/root credentials over a slow-running cron job).
    
- **Spot multi-finding chains:** Combine low-impact or isolated issues that form an escalation path together—for example, a root cron job running every five minutes paired with a world-writable script it executes.
    
- **Use automated and compliance tools as guides:** Tools like LinPEAS, WinPEAS, PowerUp, Nessus, or Lynis (and failed CIS Benchmark checks) help surface and prioritize leads, but manual verification is still needed to interpret results and avoid false positives.