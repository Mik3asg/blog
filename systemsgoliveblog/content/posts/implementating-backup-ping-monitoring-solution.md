---
title: "Implementing a Backup Ping Monitoring Solution with a Bash Script for Virtual Machines"
date: 2024-09-10T16:44:01+01:00
draft: false
tags: ['Monitoring', 'Ping', 'Connectivity', 'Alert', 'Incident']
---
# Problem Statement 
On Friday, September 6th at 21:31, we received an alert from LogicMonitor, our cloud monitoring service, indicating that one of our production web app servers (Tomcat#3) was down. Receiving such an alert late on a Friday is never ideal. After SSH-ing into the server, I confirmed it was still operational, pointing to an issue with the monitoring system rather than the VM.

# Analysis
A ticket was raised with our IT service provider, who manages our cloud workloads and VMs. Their investigation revealed:

1. Ping data stopped at 16:09 the same day.
2. All other metrics reported normally, except for ping.
3. Other servers in the same environment (e.g., Tomcat#1) were successfully reporting ping data.

Their investigation concluded that the LogicMonitor collector responsible for ping monitoring had encountered an issue, causing the false alert. After they resolved the problem, ping monitoring resumed as expected.

# Proposed Alternative: Backup Ping Monitoring Script
To avoid relying entirely on third-party services for ping checks, I developed a backup ping monitoring script. Running on a separate VM, it regularly pings target servers and sends email alerts if any fail to respond after multiple attempts. While it lacks advanced metrics or graphs, it provides a simple, effective fallback for basic network connectivity monitoring.

## Environment setup:
This backup ping monitoring solution was implemented and tested in an AWS environment. The setup included 4 EC2 instances:
- 1 EC2 instance as the `ping-monitoring server`.
- 2 EC2 instance as `target hosts`.

## Implementation steps
Steps 1 through 3 are the prerequisites before running the `ping-failure-alert.sh` script on the monitoring server.

### Step 1 - Allow ICMP Through the Firewall
- **Monitoring Server:** Allow outbound ICMP traffic (ping) for all destinations.
- **Target Servers:** Allow inbound ICMP traffic (ping), but restrict it to the monitoring server's IP.
- **Test Connectivity**: Use the ping command from the monitoring server to ensure each target server is reachable and ICMP is enabled: `ping <target_host_public_IP_address>`

### Step 2 - Install `mailx` package to enable email notification
`mailx` is a command-line email client used in Unix-like systems to send and receive emails directly from the terminal or within scripts.
- For Debian/Ubuntu: `sudo apt-get install mailx`
- For RHEL/CentOS: `sudo yum install mailx`
- For Fedora: `sudo dnf install mailx`

### Step 3 - Gmail Configuration for SMTP Authentication
To ensure that the script can send email notifications using Gmail’s SMTP service, you need to configure Gmail accordingly. Follow these steps:

#### Enable App Passwords for Gmail:
If you have Two-Step Verification enabled:
- Go to `Google Account > Security > Signing in to Google, App Passwords`
- Type a name in the `App name` field to create an App Password 

#### Configure Generate App password in `/etc/mail.rc` on the Monitoring Server
```bash
set smtp=smtps://smtp.gmail.com:465
set smtp-auth=login
set smtp-auth-user=<your_email_id>@gmail.com            # provide the main Gmail address
set smtp-auth-password=<your_generated_app_password>    # do not leave any spaces between characters
set ssl-verify=ignore
```

### Step 4 - Ping Monitoring Script
The bash script below pings the servers and sends alerts via email if they are unreachable.
- Ensure the script is executable (`chmod +x ping-failure-alert.sh`).
- Restrict access to `root` only for security.

```bash
#################################################################################
# Script Name: ping-failure-alert.sh
# Description: This script pings a predefined list of server IP addresses to check
#              their network connectivity. If any servers fail to respond after
#              a specified number of attempts and interval, an email notification
#              is sent.
# Author: Mickael Asghar
# Created on: 07/06/2024
# Updated on: 07/06/2024
#################################################################################

#!/bin/bash

# Email Configuration - Define email addresses here
recipient_email="ping-monitoring@abc.com"  # Define the primary recipient
cc_recipients=("contact1@abc.com" "contact2@abc.com)  # Define CC recipients

# Convert CC recipients array to a comma-separated string
cc_list=$(IFS=','; echo "${cc_recipients[*]}")

# Associative array of server IP addresses and their hostnames
declare -A ping_targets=(
    ["192.168.1.202"]="target_host#1"    # adjust IP address and hostname
    ["192.168.1.203"]="target_host#2"    # adjust IP address and hostname
    ["192.168.1.204"]="target_host#3"    # adjust IP address and hostname
)

# Retry settings
retry_count=3       # Number of retry attempts
retry_interval=30   # Interval in seconds between retries

# Initialize a variable to store failed hosts
failed_hosts=""

# Function to ping a host
ping_host() {
    local ip=$1
    ping -c 1 $ip > /dev/null 2>&1
    return $?
}

# Loop through each target and attempt to ping
for ip in "${!ping_targets[@]}"  # Loop over keys of the associative array
do
    hostname=${ping_targets[$ip]}  # Assign hostname from the associative array
    success=false

    for attempt in $(seq 1 $retry_count)
    do
        echo "Pinging $hostname ($ip) (Attempt $attempt of $retry_count)..."
        if ping_host $ip; then
            echo "$hostname ($ip) is reachable."
            success=true
            break
        else
            echo "$hostname ($ip) is not reachable. Waiting $retry_interval seconds before retrying..."
            sleep $retry_interval
        fi
    done

    if ! $success; then
        current_datetime=$(date "+%d/%m/%Y %H:%M:%S")  # Get the current date and time
        echo "$hostname ($ip) failed to respond after $retry_count attempts."
        failed_hosts+="Date: $current_datetime - $hostname ($ip).\n"  # Append formatted string
    fi
done

# Check if any host failed to respond and send an email if so
if [ ! -z "$failed_hosts" ]; then
    message="The following hosts failed to respond after $retry_count attempts with a $retry_interval second interval between attempts:\n$failed_hosts"
    echo -e "$message" | mailx -s "[ALERT] - Ping Failure Notification" -c "$cc_list" "$recipient_email"
else
    echo "All hosts responded successfully after $retry_count attempts."
fi
```

### Key Settings:
- **Retry Attempts (retry_count):** The script will try to ping each server 3 times before declaring it unreachable.
- **Interval Between Retries (retry_interval):** There is a 30-second interval between retries. This ensures that short downtimes (e.g., server reboot) will not trigger immediate false alerts.
- **Adjust values accordingly to meet your requirements**: `recipient_email`, `cc_recipients`, `target_host_IP`, `target_hostname`


### Step 5 - Set up the Script as a Cron Job
To make the script run automatically at regular intervals, we set it up as a cron job. In this case, I recommend running the script every 5 minutes, which provides frequent monitoring while avoiding unnecessary load.

1. Open Crontab:
```bash
crontab -e
```
2. Add the Cron Job: Add the following line to run the script every 5 minutes:
```bash
*/5 * * * * /path/to/ping-failure-alert.sh
```
3. Replace ``/path/to/ping-failure-alert.sh`` with the actual path to your script.

### Avoiding Unnecessary Alerts
With 3 retry attempts and a 30-second interval, the script waits about 90 seconds before declaring a server unreachable and sending an alert. Since the cron job runs every 5 minutes, this allows time for brief downtimes or scheduled reboots, avoiding unnecessary alerts for short interruptions like maintenance.


## Conclusion
Since ping data collection can fail unexpectedly, this backup ping monitoring script acts as a reliable fallback for basic connectivity checks. It operates independently of the main service, is lightweight, and can be customized as needed:
- Independent of the primary system.
- Minimal resource usage.
- Customizable retry limits, intervals, and email alerts.