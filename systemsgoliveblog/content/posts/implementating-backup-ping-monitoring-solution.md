---
title: "Implementing a Backup Ping Monitoring Solution with a Bash Script for Virtual Machines"
date: 2024-09-10T16:44:01+01:00
draft: true
tags: ['Monitoring', 'Ping', 'Connectivity', 'Alert', 'Incident']
---
## Problem Statement 
On Friday, September 6th at 21:31, we received an alert from LogicMonitor, our cloud monitoring service, indicating that one of our production web app servers (Tomcat#3) was down. Receiving such an alert late on a Friday is never ideal. After SSH-ing into the server, I confirmed it was still operational, pointing to an issue with the monitoring system rather than the VM.

## Analysis
A ticket was raised with our IT service provider, who manages our cloud workloads and VMs. Their investigation revealed:

1. Ping data stopped at 16:09 the same day.
2. All other metrics reported normally, except for ping.
3. Other servers in the same environment (e.g., Tomcat#1) were successfully reporting ping data.

Their investigation concluded that the LogicMonitor collector responsible for ping monitoring had encountered an issue, causing the false alert. After they resolved the problem, ping monitoring resumed as expected.

## Proposed Alternative: Backup Ping Monitoring Script
To avoid relying solely on a third-party service for critical ping checks, I created a backup ping monitoring script that runs on a separate virtual machine. This script regularly pings target servers and sends email alerts if a server fails to respond after multiple attempts.

This solution, while lacking advanced metrics or graphs, ensures basic network connectivity monitoring, especially if the main system fails.

### Gmail Configuration for SMTP Authentication
To ensure that the script can send email notifications using Gmail’s SMTP service, you need to configure Gmail accordingly. Follow these steps:

1. Enable App Passwords for Gmail:
If you have Two-Step Verification enabled:
Go to `Google Account Security`.
Under `Signing in to Google`, enable `App Passwords`.
Create an App Password for Mail.
Use the generated password in the script (`/etc/mail.rc`) as follows:
```bash
set smtp=smtps://smtp.gmail.com:465
set smtp-auth=login
set smtp-auth-user=<your_email_id>@gmail.com
set smtp-auth-password=<your_generated_app_password>
set ssl-verify=ignore
```
2. Allow Less Secure Apps (Optional, Not Recommended for Two-Step Accounts):
If you don't use Two-Step Verification, enable Less Secure Apps from your Google Account settings.
Once you have the Gmail SMTP authentication set up, you are ready to configure the script and its settings.

### Ping Monitoring Script
The bash script below pings the servers and sends alerts via email if they are unreachable.

```bash
#################################################################################
# Script Name: ping-failure-alert.sh
# Description: This script pings a predefined list of server IP addresses to check
#              their network connectivity. If any servers fail to respond after
#              a specified number of attempts and interval, an email notification
#              is sent.
#
# Pre-requisites:
#   - Install the 'mailx' package to enable email notifications.
#       For Debian/Ubuntu: sudo apt-get install mailx
#       For RHEL/CentOS: sudo yum install mailx
#       For Fedora: sudo dnf install mailx
#   - Ensure ICMP (Internet Control Message Protocol) is allowed through the
#     firewall on both this server (for outgoing pings) and on the target servers
#     (for incoming pings).
#   - SMTP Configuration: Configure the SMTP settings in '/etc/mail.rc' to use Gmail's SMTP server:
#       # Set SMTP to use Gmail's SMTP server
#       set smtp=smtps://smtp.gmail.com:465
#       set smtp-auth=login
#       set smtp-auth-user=<your_email_id>@gmail.com
#       set smtp-auth-password=<app_password_from_gmail_security>
#       set ssl-verify=ignore
#       set nss-config-dir=/etc/pki/nssdb
#   - Update the recipient email addresses and target/client IP addresses in the
#     script to match your specific requirements.
#   - Before running this script, test that each target server is reachable via
#     the ping command to confirm that ICMP is enabled on the target servers.
#       Example: ping -c 1 192.168.1.202
#
# Usage:
#   - Provide the recipient email addresses in the script
#   - Run script using: ./ping-failure-alert.sh
#
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
    ["192.168.1.202"]="Server 01"
    ["192.168.1.203"]="Server 02"
    ["192.168.1.204"]="Server 03"
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
- **Pre-requisites:** Check the instructions provided in the comment section in the top of the script.
- **Retry Attempts (retry_count):** The script will try to ping each server 3 times before declaring it unreachable.
- **Interval Between Retries (retry_interval):** There is a 30-second interval between retries. This ensures that short downtimes (e.g., server reboot) will not trigger immediate false alerts.

### Benefits:
- **Independent Monitoring:** The script operates independently of the primary monitoring system, providing a backup solution for checking server connectivity.
- **Lightweight:** It runs on a lightweight virtual machine and does not interfere with production workloads.
- **Customisable:** Retry limits, ping intervals, and recipient emails can easily be modified to fit specific needs.

### Set up the Script as a Cron Job
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
While LogicMonitor provides robust monitoring and visualization, it can occasionally fail due to issues with ping data collection, as we experienced. By implementing this backup ping monitoring script, we ensure basic connectivity checks independent of our primary monitoring service. This solution is easy to set up, lightweight, and serves as a safeguard in case of future monitoring system failures.

