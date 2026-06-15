# Threat Detection Lab: Cowrie Honeypot + Splunk SIEM on AWS

![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Splunk](https://img.shields.io/badge/splunk-000000?style=for-the-badge&logo=splunk&logoColor=white)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)

## Objective
Built an AWS lab to simulate end-to-end threat detection by deploying a Cowrie SSH honeypot in docker, forwarding live attack logs via Splunk Universal Forwarder and ingesting the JSON logs into a Splunk SIEM for real time analysis.

## Architecture Flow

<img width="801" height="541" alt="AWS-Honeypot-SIEM drawio" src="https://github.com/user-attachments/assets/461c357d-a68f-416c-a334-4d87b5780f42" />
---

## Tools & Technologies

| Role | Tool / Technology |
|---|---|
| Cloud Infrastructure | AWS EC2, AWS VPC, Security Groups (Firewalls) |
| SIEM | Splunk Enterprise, Splunk Universal Forwarder on AWS EC2 |
| Honeypot | Cowrie Docker on AWS EC2 |
| Attack | Real world Attackers/Botnets |
| Containerization | Docker |

---
## Phase 1: Building the SIEM (Splunk) om EC2 Instance 1
1. Install Splunk:
```bash
  wget -O splunk-9.2.2-d76edf6f0a15-linux-2.6-amd64.deb "https://download.splunk.com/products/splunk/releases/9.2.2/linux/splunk-9.2.2-d76edf6f0a15-linux-2.6-amd64.deb"
  sudo dpkg -i splunk-9.2.2-d76edf6f0a15-linux-2.6-amd64.deb
  sudo /opt/splunk/bin/splunk start --accept-license
  sudo /opt/splunk/bin/splunk enable boot-start
```
2. AWS Security Group Configuration:
```bash
Inbound TCP 22 (My IP) - SSH login to instance.
Inbound TCP 8000 (My IP) - Splunk Web UI login.
Inbound TCP 9997 (AWS Subnet 172.31.0.0/16) - Splunk Forwarder Receiver.
Enable Receiver: Log into Splunk UI, navigate to Settings > Forwarding and Receiving, and open port 9997.
```

---

## Phase 2: Deploying Cowrie Honeypot on EC2 Instance 2
1.  Change SSH connection to port 55555 to allow the honeypot to capture port 22. Admin login will be done using port 55555
```bash
sudo vim /etc/ssh/sshd_config
```
2.AWS Security Group Configuration:
```bash
Inbound TCP 5555 (My IP) - SSH login to instance.
Inbound TCP 22 (Anywhere (0.0.0.0)) - Honeypot SSH
```
3. Cowrie Setup
```bash
sudo apt update && sudo apt install docker.io docker-compose-v2 -y
sudo systemctl enable docker
sudo systemctl start docker
git clone https://github.com/cowrie/cowrie.git
cd cowrie
mkdir logs
sudo vim compose.yaml
```
4. compose.yaml file containing configuration for both containers:
```bash
services:
  cowrie:
    image: cowrie/cowrie:latest #get latest cowrie image
    container_name: cowrie_honeypot #name
    ports:
      - "22:2222" #forward port 22 to 2222 honeypot port
    volumes:
      - ./logs:/cowrie/cowrie-git/var/log/cowrie #logs at host:logs in container
    restart: always

  forwarder:
    image: splunk/universalforwarder:latest # get latest splunk forwarder image
    container_name: splunk_forwarder #name
    user: root 
    environment:    #splunk arguments to be passed
      - "SPLUNK_GENERAL_TERMS=--accept-sgt-current-at-splunk-com"
      - "SPLUNK_START_ARGS=--accept-license"
      - "SPLUNK_FORWARD_SERVER=172.31.XX.XX:9997"
      - "SPLUNK_PASSWORD=pas***"
    volumes: 
      - ./logs:/mnt/cowrie_logs:ro #logs at host:logs in container
    restart: always
```
5. Start both containers
```bash
sudo docker compose up -d
```

---

## Phase 3: Connecting Splunk Forwarder container to Splunk
```bash
#add splunk instance IP to forwarder
sudo docker exec -it splunk_forwarder /opt/splunkforwarder/bin/splunk add forward-server 172.31.XX.XX:9997 -auth "admin:pas***"
#add the log file path and set soure type json for splunk parsing
sudo docker exec -it splunk_forwarder /opt/splunkforwarder/bin/splunk add monitor "/mnt/cowrie_logs/cowrie.json" -sourcetype _json -auth "admin:pas***"
##check if forwarding is active
sudo docker exec -it splunk_forwarder /opt/splunkforwarder/bin/splunk list forward-server -auth "admin:pas***"
```
`Active forwards:
172.31.XX.XX:9997
Configured but inactive forwards:
None`

---

## Phase 4: Threat Intelligence & Incident Response (Real Time Data on Splunk UI)
The following queries and dashboards represent live, real-time attacks captured by the honeypot and routed into Splunk.

### 1. Successful Logins
By tracking successful authentication events, we can immediately isolate compromised sessions where an attacker bypassed the outer perimeter and gained execution rights.
```splunk
sourcetype="_json" eventid="cowrie.login.success" | table _time, src_ip, username, password
```
<img width="1851" height="696" alt="success_login" src="https://github.com/user-attachments/assets/bc4d7ad6-5255-4fd4-b6c0-c73247c81f16" />

### 2. Dropped Files/Malware
```splunk
sourcetype="_json" eventid="cowrie.session.file_upload" | table _time, src_ip, shasum, filename
```
<img width="1866" height="404" alt="droppedfiles" src="https://github.com/user-attachments/assets/a4be9312-97c6-436a-a8a2-c00597cf10b7" />

### 3. OSINT & Malware Analysis (IPLocation & VirusTotal)

Extracting the `shasum` (SHA-256 hash) and `src_ip` from the Splunk logs allows us to pivot to OSINT tools to identify the threat actors and their tools.

* **Threat Actor 1:** `101.96.203.55` (Origin: China)

<img width="600" height="300" alt="ipchina" src="https://github.com/user-attachments/assets/25bd34aa-633c-4f90-aa64-a1d0a98fb694" />

* **Threat Actor 2:** `83.236.176.109` (Origin: Germany)

<img width="600" height="300" alt="ipgermany" src="https://github.com/user-attachments/assets/1d86933f-62e7-4f33-b3d3-549bd8f49cff" />

**VirusTotal Findings:**
Analysis confirms both dropped payloads are variants of **Cryptominer Trojans** disguised under legitimate-sounding system names like `sshd` or `discord-exploit` to evade detection.

*File 1 Origin (China):*

<img width="600" height="150" alt="miner-china" src="https://github.com/user-attachments/assets/240a9da3-517a-4aa3-ab15-5161f7d1a423" />
<img width="900" height="250" alt="miner-china2" src="https://github.com/user-attachments/assets/bd5256d5-b6e3-4c38-bc84-1d46769e0854" />

*File 2 Origin (Germany):*

<img width="600" height="150" alt="discord-sshd-coinminer-de" src="https://github.com/user-attachments/assets/80e9f2ec-1dbf-4f7c-9db9-1e76e09b69bd" />
<img width="900" height="250" alt="discord-sshd-coinminer-de2" src="https://github.com/user-attachments/assets/d4ab7955-aa7e-49e1-996e-d0c9975f9c50" />

### 4. Terminal Inputs after logins
```bash
sourcetype="_json" eventid=cowrie.command.input | table _time, src_ip, input
```
<img width="1865" height="1000" alt="input_splunk1" src="https://github.com/user-attachments/assets/e1e10b47-ed2e-4628-8561-e2830d47ffbd" />
<img width="868" height="350" alt="input_splunk2" src="https://github.com/user-attachments/assets/3911ca30-c962-4da3-8244-4405c0700920" />


Identified four different categories of behavior hitting the honeypot:

1. Botnets successfully brute-forcing credentials and immediately running `uname -a` to know the OS details before downloading a specific malware payload.
2. Attackers dropping malicious binaries into hidden folders(`./48867..`), renaming them to blend in with normal system processes (`sshd`), granting execution permissions (`chmod +x`) and running them silently in the background(`nohup`).
3. Sophisticated scripts scanning `/proc/cpuinfo` and `lspci` to check for NVIDIA graphics cards,to determine if the compromised server is profitable for mining cryptocurrency.
4. Slower and standard commands of Linux navigation (`whoami`, `pwd`, `ls`) indicates manual exploration by a human. *(This interaction was performed by me to validate the pipeline).*

---
