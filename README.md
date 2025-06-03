# Detecting SSH Brute-Force with Azure Sentinel. SSH hardening.

### Description

SSH (Secure Shell) is a network protocol often used to facilitate network connections to remote servers. It is used in the tasks of remotely managing infrastructure, accessing services in the cloud, bypassing firewall restrictions, transferring files and so on. Due to the functionality of the service, attackers often aim to exploit it. In this walkthrough, I've utilized the capabilities of Azure Sentinel to detect Brute-Force attacks and implemented SSH hardening techniques to minimize risk.

Note: Setup of VMware software, subsequent installation of the OS on the machines, creation of Azure account/Subscription/Log analytics workspace and Sentinel instance are not the subject of this walkthrough, so information about them will not be included. Nonetheless, articles that explain the processes can be found below.

Creating Azure account: https://azure.microsoft.com/en-us/pricing/purchase-options/azure-account
Creating subscriptions and licenses: https://learn.microsoft.com/en-us/microsoft-365/enterprise/subscriptions-licenses-accounts-and-tenants-for-microsoft-cloud-offerings?view=o365-worldwide
Creating Log Analytics Workspace: https://learn.microsoft.com/en-us/azure/azure-monitor/logs/quick-create-workspace?tabs=azure-portal
Deployment for Azure Sentinel: https://learn.microsoft.com/en-us/azure/sentinel/deploy-overview
Setting up VMs in VMware: https://www.instructables.com/How-to-Setup-a-Virtual-Machine-in-VMware-Workstati/

.iso files for the VMs can be found on the internet.

##### Machines/Environments used:
- Ubuntu 24.04.02 LTS / IP:192.168.1.149
- Windows 10 22H2 / IP:192.168.1.221
- Kali Linux 2025.1 / IP: 192.168.1.116

##### Software used:
- VMware Workstation 17.6.3 for the above machines. Network adapter set to "Bridged (Automatic)"
- Azure Sentinel / Azure Arc / Azure Monitor Agent
- Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak
- OpenSSH_9.6p1 Ubuntu-3ubuntu13.11, OpenSSL 3.0.13 30


##### Steps in the process:
1. ~~Install virtualization software and OSs~~
2. ~~Create an Azure account and deploy Azure Sentinel~~


We will continue with the deployment of Azure Arc. The Ubuntu machine we have is considered non-native for Azure. Such machines must be onboarded to Azure Arc to be visible in the Azure Portal. Note that the possibility of adding the machine with the Legacy OMS agent is not available to us, as the OMS agent [does not support](https://github.com/microsoft/OMS-Agent-for-Linux) newer Ubuntu versions such as 24.04.02 LTS. 

##### 3. Adding the machine to Azure Arc
- We need to add the machine as a resource. We do this by generating a script via the Azure Portal
-![](Images/Screenshot%202025-06-03%20224541.png)
![](Images/Screenshot%202025-06-03%20225345.png)
![](Images/Screenshot%202025-06-03%20225406.png)![](Images/Screenshot%202025-06-03%20225646.png)

- After we generate the script we have two options - to download it as a shell script file and to copy-paste it to create our own shell script. For the purpose of the demo, we download it. Next, we have to run it as sudo in the Ubuntu environment.
![](Images/Screenshot%202025-06-03%20235445.png)
- We must then login via a web browser to authenticate ourselves. The login credentials used are the ones for your Azure Account
![](Images/Screenshot%202025-06-03%20235621.png)- After authentication is done, terminal should confirm the machine is connected to Azure. We ourselves can confirm that by reviewing the resources under Azure Arc![](Images/Screenshot%202025-06-03%20235752.png)
![](Images/Screenshot%202025-06-03%20235927.png)

##### 4. Creating a DCR (Data Collection Rule) - In order to ingest logs to our log analytics workspace, we must create a DCR. This is done by going to Data Collections Rule on Azure.![](Images/Screenshot%202025-06-04%20000229.png)
- It is considered good practice to make the DCR the same region as your machine. Make sure to pick the correct platform for the DCR. ![](Images/Screenshot%202025-06-04%20000337.png)
- Next, add the machine as a resource for the DCR and provide the log analytics space to which you would like for the logs to be ingested.
![](Images/Screenshot%202025-06-04%20000443.png)
![](Images/Screenshot%202025-06-04%20000513.png)
![](Images/Screenshot%202025-06-04%20000548.png)
![](Images/Screenshot%202025-06-04%20000632.png) 
- After the creation, you should see a page that indicates the deployment is complete.![](Images/Screenshot%202025-06-04%20000720.png)
##### 5. Install the Syslog data connector from the Content Hub
- [This is a straightforward process.](https://learn.microsoft.com/en-us/azure/sentinel/cef-syslog-ama-overview?tabs=single#setup-process-to-collect-log-messages)  . It gives us out-of-the-box analytics rules we will use. It is important to note that the creation of DCR can be made from the connector page itself, and the AMA agent is automatically installed. For the purpose of the demo, we choose to do it by hand. ![](Images/Screenshot%202025-06-04%20001240.png)
- You can now verify that logs are ingested from the machine by querying your log analytics workspace. We see that we have a heartbeat and syslogs are being ingested.![](Images/Screenshot%202025-06-04%20001108.png)


#### <p align="center" >SSH Brute-Forcing, detecting and hardening.</p>


During the installation of Ubuntu, you are asked if you want the modules for usage of SSH protocol to be installed, to which I clicked yes. This saves us some time. Let's test if SSH is working.

Confirmed working.
![](Images/Screenshot%202025-06-04%20004522.png)

Now, let's play around with [Hydra](https://www.kali.org/tools/hydra/) on the Kali Linux machine. Hydra is a network tool that is used to brute-force different services. We will use it to try and brute-force the SSH service on the Ubuntu machine.  Hydra takes different arguments that can indicate multiple things. The main ones we will use are arguments that provide us with the ability to indicate a list of users and a list of passwords that should be tried against the SSH service on the Ubuntu machine. I use the list of users provided in the SecLists repository by [Daniel Miessler](https://github.com/danielmiessler), and a randomPasswords.txt file with passwords filled by me, which contains the correct password for the Ubuntu machine.

Success, we managed to obtain the credentials for the machine, and we can use them to log in.
![](Images/Screenshot%202025-06-04%20010039.png)

In a perfect scenario, there are multiple things that can be done in order to prevent the negative outcome of getting your server accessed by an unauthorized person. For example, use stronger passwords. There is a free [ssh-audit tool](https://github.com/jtesta/ssh-audit) that will audit the security posture of your SSH, so you can have better view of the aspects that needs improving. Common ways to secure SSH is to change the port for SSH, reduce the number of connection attempts per unit of time, setup 2FA and more. For the purpose of the demo, we will disable password login and use key-based authentication. We will use two keys, Public (stored on the SSH server) and private (stored on the machine that connects to the server, in our case, the Windows Machine).  

