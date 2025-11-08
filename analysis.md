# Analysis

## Overview
- Originally I hadn't planned to document real-world logs, as I didn't expect my honeypot to receive any traffic. However, coincidentally, while I was writing this report, **several** alerts appeared in Sentinel indicating SSH brute-force attempts.
- This part of the report will be quite short as the honeypot was only running for a **limited period of time** and the overall goal wasn't to do a full-scale investigation. 

### Statistics
- Over a combined uptime of approximately **15** hours, the honeypot recorded **429** SSH login attempts, including **4** successful logins.
![real](/assets/images/realog.jpg)

> Note: I added additional code to exclude login attempts originating from my location to ensure data accuracy. The obscured code in the image reflects this adjustment.

![successfulssh](/assets/images/ip.jpg)

- These 429 attempts originated from **6** different countries, with **India** having the **highest** number of attempts.
![countries](/assets/images/countries.jpg)

> Note: These geolocation results **may not** be fully accurate due to the possible use of VPNs, proxies or any means of anonymisation by attackers.

## Verification with Third-Party Threat Intelligence
- To further validate the malicious nature of the activity, I cross-checked several attacking IP addresses using **VirusTotal**.
- For example, the IP address **`167.71.6.212`** was flagged as malicious by 4 different security vendors which further supports the evidence that these were genuine brute-force attempts.
![virustotal](/assets/images/virustotal.png)

## Attacker Behaviour
- Next I **analysed** the **behaviour** of the attackers who successfully logged in to the honeypot
- The screenshot below shows several of the commands executed during their sessions.
![cmd](/assets/images/cmd.jpg)

- As you can see many of these commands were repetitive across different IP addresses (more on this later). This may suggest that the attackers were utilising automated scripts.
- Below are explanations for some of the most common commands:
1. `uname -s -v -n -m 2 > /dev/null`
  - Gathers **basic system information** such as the kernel name/version, hostname and machine architecture
  - The `2 > /dev/null` hides any error output to avoid cluttering the terminal and provides a cleaner output.

2. `uname -m 2 > /dev/null`
  - Displays only the machines **hardware architecture**.
  - Attackers may use this to determine which binary executables are compatible with the system.
  
3. `cat /proc/uptime 2 > /dev/null | cut -d. -f1`
  - Reads systems **uptime** (in seconds) from `/proc/uptime`.
  - The cut command trims off the decimal part, leaving a whole number.
  - Uptime information may help attackers estimate how long the system has been running - very short uptime could indicate a recently deployed VM.

4. `export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:$PATH uname=$(uname -s -v -n -m 2>/dev/null) arch=$(uname -m 2>/dev/null) uptime=$(cat /proc/uptime 2>/dev/null | cut -d. -f1) cpus=$( (nproc || grep -c "^processor" /proc/cpuinfo) 2>/dev/null | head -1) cpu_model=$( (grep -m1 -E "model name|Hardware" /proc/cpuinfo | cut -d: -f2- | sed 's/^ *// s/ *$//' lscpu 2>/dev/null | awk -F: '/Model name/ {gsub(/^ +| +$/,"",$2) print $2 exit}' dmidecode -s processor-version 2>/dev/null | head -n1 uname -p 2>/dev/null) | awk 'NF{print exit}' ) gpu_info=$( (lspci 2>/dev/null | grep -i vga lspci 2>/dev/null | grep -i nvidia) 2>/dev/null | head -n50) cat_help=$( (cat --help 2>&1 | tr '\n' ' ') || cat --help 2>&1) ls_help=$( (ls --help 2>&1 | tr '\n' ' ') || ls --help 2>&1) last_output=$(last 2>/dev/null | head -n 10) echo "UNAME:$uname" echo "ARCH:$arch" echo "UPTIME:$uptime" echo "CPUS:$cpus" echo "CPU_MODEL:$cpu_model" echo "GPU:$gpu_info" echo "CAT_HELP:$cat_help" echo "LS_HELP:$ls_help" echo "LAST:$last_output"`
  - This is a comprehensive **single-line reconnaissance script** that gathers and prints **detailed system information** including CPU, GPU, uptime, login history, and command help outputs; by checking how common commands respond, attackers can sometimes detect honeypots.
  - Such scripts are commonly used to determine whether the compromised system is real, virtualised or monitored, and to identify **suitable** payloads for further exploitation.

---

### Analyis
- Based on the data, it is highly likely that these attacks were part of an automated botnet campaign, rather than targeted human activity.
- Many malicious botnets continuously scan public IP ranges for open services such as SSH/FTP.
- Once an open service is found, automated tools attempt to brute-force login credentials using large dictionaries of common usernames/passwords.
- If successful, these bots typically run short **info-gathering scripts** (like the ones above) to collect system info and then move on to: report back to a **command-and-control (C2)** server or to prepare the host for malware deployment, privilege escalation or lateral movement. 
