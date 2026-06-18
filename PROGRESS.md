Learning Progress

What I've Done So Far

### Day 1 - Built Azure Infrastructure

I created my first cloud infrastructure on Azure. Set up a resource group called devops-lab-rg and built a virtual network with the address space 10.0.0.0/16. Added a network security group to control traffic. Then deployed an Ubuntu 22.04 Linux VM and downloaded the SSH key. Connected to the VM successfully using SSH from Windows.

The VM is in Australia East region. I have a public IP assigned so I can access it from anywhere.

### Day 2 - Learned Linux and Bash

Started exploring the Linux file system. Went to root (/), checked out /home, and navigated to my home directory. Found my .ssh folder where the SSH authentication keys are stored.

Ran some basic Linux commands to understand the system:
- ls -la to list files with permissions
- df -h to see disk space (29GB total, only 6% used)
- free -h to check memory (7.7GB available)
- ps aux to see what processes are running
- nproc to count CPUs (2 cores)
- uptime to see how long the VM has been running
- hostname to get the server name (devops-vm)
- whoami to confirm I'm logged in as azureuser

Then I started learning bash scripting. Created hello.sh which was a simple script that printed out the hostname, date, and current user. The key thing I learned was command substitution - using $(command) to run a command inside the text.

Created a second script called systeminfo.sh that's more useful. It displays system information - disk space, memory, CPU count, and uptime. This script shows why automation matters. Instead of typing 5 different commands manually every time you want to check a server, you run one script.

## Commands I Learned

ls - list files
ls -la - list all files with details and permissions
cd - change directory
cd / - go to root
cd ~ - go to home
pwd - show current directory
df -h - show disk space
free -h - show memory usage
nproc - show number of CPUs
uptime - show how long system is running
hostname - show server name
whoami - show current user
date - show current date and time
cat > file - create a new file
chmod +x - make a file executable
./script - run a script

## Key Things I Understood

Bash scripting is just running multiple commands in sequence. Instead of typing them one by one, you write them in a file and execute them all at once.

The $(command) syntax is powerful - it runs a command and puts the result into text. So $(hostname) actually runs the hostname command and inserts the server name.

Piping with | takes the output of one command and sends it to another command. Used this to filter output.

Linux has a file system hierarchy - / is root, /bin is where commands are, /home is where users live, /etc is configuration, /var is for logs and data.

## File Structure

The infrastructure I built:
- Azure resource group: devops-lab-rg
- Virtual network: devops-vnet with subnet 10.0.0.0/16
- Network security group: devops-nsg for firewall rules
- Linux VM: devops-vm running Ubuntu 22.04
- Public IP: 20.5.184.90 for external access
- SSH key: devops-vm_key.pem for authentication

## Scripts Created

hello.sh - prints welcome message with hostname, date, and user
systeminfo.sh - displays disk space, memory, CPU count, and system uptime

Both are stored in the home directory and are executable.

## Why This Matters

Automation is the core of DevOps. A manual task that takes 10 minutes becomes a 30-second script. When you have 50 servers, that's huge time savings.

The infrastructure I built is real production-like setup. VNets, security groups, VMs - these are exactly what companies use in the cloud.

Linux knowledge is essential because most servers run Linux, not Windows. Learning bash scripting is learning the language of server automation.

## What's Next

Going to learn file permissions and understand chmod. Then user management. After that package management with apt. Each of these connects to the infrastructure and automation concepts I'm building.