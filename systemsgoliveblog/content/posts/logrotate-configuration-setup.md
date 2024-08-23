---
title: "Logrotate Configuration Setup in AlmaLinux 8.9"
date: 2024-08-20T13:49:28+01:00
draft: false
tags: ['Logrotate', 'Linux', 'SysAdmin', 'DiskSpaceOptimisation', 'LogManagement']
---
# Logrotate Configuration Setup  in AlmaLinux 8.9

## **Introduction**
Before diving into the details, I want to share an overview of a recent log rotation project I worked on. Below is a breakdown of the tasks, challenges, and solutions, all while protecting confidential data. By implementing these solutions, we reduced log file disk usage from 6GB to 2GB—a 66.7% reduction.

Logrotate helps manage log files by automatically rotating, compressing, and removing them when they become too large or outdated, preventing excessive disk space usage and ensuring system stability.

## Table of Contents
- [What is logrotate in a nutshell?](#what-is-logrotate-in-a-nutshell)
- [Requirements](#requirements)
- [Version Information for Logrotate Linux Tomcat and MySQL](#version-information-for-logrotate-linux-tomcat-and-mysql)
- [Scope of logs per logrotate configuration file](#scope-of-logs-per-logrotate-configuration-file)
- [Implementation of Tomcat Log Rotation](#implementation-of-tomcat-log-rotation)
  - [Prerequisite - Disable internal Tomcat log rotation process](#prerequisite---disable-internal-tomcat-log-rotation-process)
    - [Edit `/opt/tomcat/conf/logging.properties` configuration file](#edit-opt-tomcat-confloggingproperties-configuration-file)
    - [Edit `/opt/tomcat/conf/server.xml` configuration file](#edit-opt-tomcat-confserverxml-configuration-file)
    - [Delete Unnecessary Archived Logs in `/opt/tomcat/logs/archived/` directory](#delete-unnecessary-archived-logs-in-opt-tomcatlogsarchived-directory)
  - [Logrotate Configuration for `catalina.out` log files](#logrotate-configuration-for-catalinaout-log-files)
  - [Logrotate Configuration for Tomcat Miscellaneous Logs](#logrotate-configuration-for-tomcat-miscellaneous-logs)
- [Logrotate Configuration for `httpd` log files](#logrotate-configuration-for-httpd-log-files)
- [Logrotate Configuration for `unison.log` file](#logrotate-configuration-for-unisonlog-file)
  - [Changing the SELinux Context for `/root/unison.log`](#changing-the-selinux-context-for-rootunisonlog)
  - [Custom cron job for unison log](#custom-cron-job-for-unison-log)
- [Logrotate Configuration for MySQL-related log files](#logrotate-configuration-for-mysql-related-log-files)
  - [Configuring MySQL Authentication for Log Rotation](#configuring-mysql-authentication-for-log-rotation)
- [Logrotate Troubleshooting and Best Practices](#logrotate-troubleshooting-and-best-practices)
  - [Refer to Linux Manual Page](#refer-to-linux-manual-page)
  - [Maintain Root Ownership for Logrotate Configuration Files](#maintain-root-ownership-for-logrotate-configuration-files)
  - [Verify Ownership Permission and SELinux settings of log files](#verify-ownership-permission-and-selinux-settings-of-log-files)
  - [Understanding the ‘su’ Directive with Specific ‘create’ Settings](#understanding-the-su-directive-with-specific-create-settings)
  - [Run Manual Test](#run-manual-test)
  - [Debug Mode](#debug-mode)
  - [Last Log Rotation Timestamps by Logrotate](#last-log-rotation-timestamps-by-logrotate)
  - [Compression Issue](#compression-issue)
  - [Verify Full File Paths](#verify-full-file-paths)
  - [Check for Typos](#check-for-typos)
  - [Curly Braces and Comments](#curly-braces-and-comments)

## What is logrotate in a nutshell?

Logrotate is designed to ease the administration of systems that generate large numbers of log files. It allows automatic rotation, compression, removal, and mailing of log files. Each log file may be handled daily, weekly, monthly, or when it grows too large.

Source: [https://linux.die.net/man/8/logrotate](https://linux.die.net/man/8/logrotate)

## Requirements

Implement custom logrotate configurations for Tomcat and MySQL DB virtual servers for disk space optimisation. To do this, we define specific directives in the logrotate configurations, including:
- Setting the occurrence of log rotation (e.g., daily) for timely rotation of log files.
- Specifying the retention policy for rotated log files (e.g., retaining the last 10 rotations).
- Leveraging the compress directive.
- Configuring the ownership and permissions for the rotated log files to ensure appropriate access and security.

## Version Information for Logrotate Linux Tomcat and MySQL

The logrotate configuration detailed in this document has been developed and tested on the following system versions. Knowing the version compatibility across these components is crucial to ensure that everything functions correctly and performs optimally.

| Service/Tool | Command                              | Output                                 | Date          |
|--------------|--------------------------------------|----------------------------------------|---------------|
| Logrotate    | `logrotate --version`                | logrotate 3.14.0                       | 12/08/2024    |
| Linux        | `cat /etc/os-release`| AlmaLinux 8.9                          | 12/08/2024     |
| Tomcat       | `/opt/tomcat/bin/version.sh`         | Apache Tomcat/9.0.83                   | 12/08/2024     |
| MySQL        | `mysql -V`                           | mysql Ver 8.0.37 for Linux on x86_64 (MySQL Community Server - GPL) |12/08/2024 |

## Scope of logs per logrotate configuration file

The table below outlines the scope of log files managed by each logrotate configuration. Complete scripts for each logrotate configuration file are provided in the subsequent sections of this document.

| Logrotate configuration          | Log files                                             |
|----------------------------------|-------------------------------------------------------|
| `/etc/logrotate.d/catalina_out`  | `/opt/tomcat/logs/catalina.out`                       |
| `/etc/logrotate.d/tomcat_misc`   | `/opt/tomcat/logs/catalina.log`, `/opt/tomcat/logs/localhost.log`, `/opt/tomcat/logs/localhost_access_log.txt`, `/opt/tomcat/logs/debug.log`, `/opt/tomcat/logs/task-debug.log`, `/opt/tomcat/logs/task-generation.log` |
| `/etc/logrotate.d/httpd`         | `/var/log/httpd/access_log`, `/var/log/httpd/error_log`|
| `/etc/logrotate.d/mysql`         | `/var/log/mysqld.log`, `/var/log/mysql.log`, `/var/lib/mysql/log_slow_query.log` |

## Implementation of Tomcat Log Rotation

### Prerequisite - Disable internal Tomcat log rotation process

As a prerequisite before configuring logrotate for the Tomcat-related log files, it is necessary to disable Tomcat's internal log rotation process to avoid any conflicts with logrotate. To do this, update the following configuration files on the Tomcat virtual machines.
- **File:** `/opt/tomcat/conf/logging.properties`  
- **File:** `/opt/tomcat/conf/server.xml`

#### Edit `/opt/tomcat/conf/logging.properties` configuration file

- Set `maxDays` to `-1`. This allows log entries to be continuously written to a single log file without a retention period as specified in the documentation.
- Set the `rotatable` parameter to `false`. This enables external tools such as logrotate to manage the log rotation cycle.
- Comment out the manager and host-manager logs as they are no longer needed.

Apply the changes in the `logging.properties` file accordingly as shown below:

```properties
1catalina.org.apache.juli.AsyncFileHandler.level = FINE
1catalina.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
1catalina.org.apache.juli.AsyncFileHandler.prefix = catalina.
# Retain log file (no deletion) on the server file system, and disable internal Tomcat log rotation,
# as log rotation and cleanup are managed externally by logrotate.
1catalina.org.apache.juli.AsyncFileHandler.maxDays = -1
1catalina.org.apache.juli.AsyncFileHandler.rotatable = false
1catalina.org.apache.juli.AsyncFileHandler.encoding = UTF-8

2localhost.org.apache.juli.AsyncFileHandler.level = FINE
2localhost.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
2localhost.org.apache.juli.AsyncFileHandler.prefix = localhost.
# Retain log file (no deletion) on the server file system, and disable internal Tomcat log rotation,
# as log rotation and cleanup are managed externally by logrotate.
2localhost.org.apache.juli.AsyncFileHandler.maxDays = -1
2localhost.org.apache.juli.AsyncFileHandler.rotatable = false
2localhost.org.apache.juli.AsyncFileHandler.encoding = UTF-8

# These logs are no longer required
#3manager.org.apache.juli.AsyncFileHandler.level = FINE
#3manager.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
#3manager.org.apache.juli.AsyncFileHandler.prefix = manager.
#3manager.org.apache.juli.AsyncFileHandler.maxDays = -1
#3manager.org.apache.juli.AsyncFileHandler.rotatable = false
#3manager.org.apache.juli.AsyncFileHandler.encoding = UTF-8

# These logs are no longer required
#4host-manager.org.apache.juli.AsyncFileHandler.level = FINE
#4host-manager.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
#4host-manager.org.apache.juli.AsyncFileHandler.prefix = host-manager.
#4host-manager.org.apache.juli.AsyncFileHandler.maxDays = -1
#4host-manager.org.apache.juli.AsyncFileHandler.rotatable = false
#4host-manager.org.apache.juli.AsyncFileHandler.encoding = UTF-8

java.util.logging.ConsoleHandler.level = FINE
java.util.logging.ConsoleHandler.formatter = org.apache.juli.OneLineFormatter
java.util.logging.ConsoleHandler.encoding = UTF-8


############################################################
# Facility specific properties.
# Provides extra control for each logger.
############################################################

org.apache.catalina.core.ContainerBase.[Catalina].[localhost].level = INFO
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].handlers = 2localhost.org.apache.juli.AsyncFileHandler

# These logs are no longer required for our project
#org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/manager].level = INFO
#org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/manager].handlers = 3manager.org.apache.juli.AsyncFileHandler

# These logs are no longer required for our project
#org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/host-manager].level = INFO
#org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/host-manager].handlers = 4host-manager.org.apache.juli.AsyncFileHandler
```
After making the changes, perform the following steps:
- Restart Tomcat service: `systemctl restart tomcat`
- Check Tomcat service status: sudo `systemctl status tomcat`
- Reboot the VM `reboot` command.


#### Delete Unnecessary Archived Logs in `/opt/tomcat/logs/archived/` directory

Logs such as `task-debug.YYYY-MM-DD.log` and `debug.YYYY-MM-DD.2.log` are generated for debugging purposes and are not needed for this project. These can be safely removed to free up disk space. 
As a temporary solution, we are using a cron job to remove the contents of this directory. To prevent these logs from being created in the future, their generation and rotation must be disabled at the application level.

To set up the cron job:
- As the root user, open the crontab editor by running the following command: `crontab -e`
- Add the following line to the crontab file:

```bash
# Daily at 12:15 AM remove all contents from '/opt/tomcat/logs/archived' as these logs are not needed.
# Disable log generation and rotation at the application level to prevent future creation.
15 0 * * 0 rm -rf /opt/tomcat/logs/archived/*
```

## Logrotate Configuration for `catalina.out` log files
1. Use SSH to access the Tomcat #01 server as the root user.
2. Navigate to the directory by entering `cd /etc/logrotate.d` in the terminal.
3. Copy the existing file `tomcat.disabled` and rename it to `catalina_out` using the command:
`cp -a tomcat.disabled catalina_out`

**Note:** It is crucial to include the -a option in the `cp` command to ensure that the new file retains the ownership, permissions, and SELinux context of the original file.


```bash
/opt/tomcat/logs/catalina.out 
{
    su tomcat adminops
    copytruncate
    daily                        
    rotate 7                    
    compress
    missingok
    notifempty
    dateext
    dateformat -%Y-%m-%d-%H%M
    create 770 tomcat adminops    
}
```
4. For testing purposes, you can perform a dry run of the log rotation process to see what would happen without actually rotating the logs: `logrotate -vd /etc/logrotate.d/catalina_out`
5. Repeat the previous steps for each additional Tomcat VM to apply the same logrotate configuration.

## Logrotate Configuration for Tomcat Miscellaneous Logs
1. Use SSH to access the Tomcat #01 server as the root user.
2. Navigate to the directory by entering cd /etc/logrotate.d in the terminal.
3. Copy the existing file tomcat.disabled and rename it to catalina_out using the command: `cp -a tomcat.disabled tomcat_misc`

**Note:** It is crucial to include the -a option in the `cp` command to ensure that the new file retains the ownership, permissions, and SELinux context of the original file.

4. Update the file with the following custom configuration

```bash
/opt/tomcat/logs/catalina.log
/opt/tomcat/logs/localhost.log
/opt/tomcat/logs/localhost_access_log.txt
/opt/tomcat/logs/debug.log
/opt/tomcat/logs/task-debug.log
/opt/tomcat/logs/task-generation.log
{
    copytruncate
    daily                        
    rotate 7                    
    compress
    missingok
    notifempty
    su tomcat tomcat
    create 640 tomcat tomcat
    sharedscripts
    postrotate
        /bin/systemctl reload tomcat.service > /dev/null 2>/dev/null || true
    endscript
}
```
5. For testing purposes, you can perform a dry run of the log rotation process to see what would happen without actually rotating the logs: `logrotate -vd /etc/logrotate.d/tomcat_misc`
6. Repeat the previous steps for each additional Tomcat VM to apply the same logrotate configuration.


## Logrotate Configuration for `httpd` log files
1. Use SSH to access the Tomcat #01 server as the root user.
2. Navigate to the directory by entering `cd /etc/logrotate.d` in the terminal.
3. Copy the existing file `tomcat.disabled` and rename it to `catalina_out` using the command:
`cp -a tomcat.disabled catalina_out`

**Note:** It is crucial to include the -a option in the `cp` command to ensure that the new file retains the ownership, permissions, and SELinux context of the original file.

4. Update the file with the following custom configuration

```bash
/var/log/httpd/access_log 
/var/log/httpd/error_log 
{
    su root adminops
    daily
    rotate 7
    compress
    missingok
    notifempty
    create 770 root adminops
    sharedscripts
    postrotate
        /bin/systemctl reload httpd.service > /dev/null 2>/dev/null || true
    endscript
}
```
5. For testing purposes, you can perform a dry run of the log rotation process to see what would happen without actually rotating the logs: `logrotate -vd /etc/logrotate.d/httpd`
6. Repeat the previous steps for each additional Tomcat VM to apply the same logrotate configuration.

## Logrotate Configuration for `unison` log files
1. Use SSH to access the Tomcat #01 server as the root user.
2. Navigate to the directory by entering `cd /etc/logrotate.d` in the terminal.
3. Copy the existing file `tomcat.disabled` and rename it to `unison` using the command:
`cp -a tomcat.disabled unison`

**Note:** It is crucial to include the -a option in the `cp` command to ensure that the new file retains the ownership, permissions, and SELinux context of the original file.

4. Update the file with the following custom configuration

```bash
# The 'daily' directive is intentionally omitted because it caused issues with the default logrotate daily cron job.
# Instead, logrotate is manually invoked via a custom cron job set to run daily at 12:45 AM.
# Cron job: 45 0 * * * /usr/sbin/logrotate /etc/logrotate.d/unison
# This ensures that unison logs are rotated daily without conflicts.

/root/unison.log 
{
    rotate 7
    copytruncate
    compress
    dateext
    dateformat -%Y-%m-%d-%H%M
    missingok
    notifempty
    create 600 root root
}
```

### Changing the SELinux Context for `/root/unison.log`
The following steps outline a workaround for the issue where logrotate cannot rotate `unison.log` due to SELinux permissions, as the file is initially located under the `/root` directory. As the root user, change the SELinux context type for the `/root/unison.log` file from `admin_home_t` to `var_log_t` using the following commands:

-	Check the current SELinux context: `ls -Z /root/unison.log`
-	Set the new SELinux context: `semanage fcontext -a -t var_log_t "/root/unison.log"`
-	Apply the context change: `restorecon -v /root/unison.log`
-	Verify the new SELinux context has been applied: `ls -Z /root/unison.log`
-	Output expected: `system_u:object_r:var_log_t:s0 /root/unison.log`

### Custom cron job for `unison.log`
Due to an issue with the `daily` directive in logrotate not working as expected, we need to set up a custom cron job to manually run logrotate for Unison logs. 
Follow these steps to create the cron job:
-	As the root user, open the crontab editor by running the following command: `crontab -e`
-	Add the following line to the crontab file to manually run logrotate for Unison logs daily at 12:30 AM:

```bash
# Manually run logrotate for unison logs daily at 12:45 AM as a workaround, since the 'daily' directive in logrotate was not working for some reason.

30 0 * * * /usr/sbin/logrotate /etc/logrotate.d/unison
```
-	Save and close the crontab file.
-	Repeat the previous steps for each additional Tomcat VM to apply the same logrotate configuration. However, schedule the daily crontab on the other Tomcat VMs for 12:45 AM. This timing ensures that the master Unison sync, which runs on the Tomcat 01 VM, completes first, allowing the other VMs to rotate their unison.log files with a slight delay.


## Logrotate Configuration for MySQL-related log files

1. Use SSH to access the Tomcat #01 server as the root user.
2. Navigate to the directory by entering `cd /etc/logrotate.d` in the terminal.
3. Open the file named mysql for editing (e.g., vim or nano).

**Important Note:** We encountered an issue where rotated MySQL log files were not being compressed as expected. To resolve this, the MySQL logrotate configuration file was updated with additional directives. The key directives added are the following:
-	delaycompress
-	dateext
-	dateformat -%Y-%m-%d-%H%M

**Outcome:** These updates have successfully addressed the compression issue with rotated log files.


4. Update the file with the following configuration:

```bash
# The log file name and location can be set in
# /etc/my.cnf by setting the "log-error" option
# in [mysqld]  section as follows:
#
# [mysqld]
# log-error=/var/log/mysqld.log
#
# For the mysqladmin commands below to work, root account
# password is required. Use mysql_config_editor(1) to store
# authentication credentials in the encrypted login path file
# ~/.mylogin.cnf
#
# Example usage:
#
#  mysql_config_editor set --login-path=client --user=root --host=localhost --password
#
# When these actions has been done, un-comment the following to
# enable rotation of mysqld's log error.
#

/var/log/mysqld.log
/var/log/mysql.log
/var/lib/mysql/log_slow_query.log
{
        daily
        rotate 5
        copytruncate
        compress
        delaycompress
        dateext
        dateformat -%Y-%m-%d-%H%M
        missingok
        notifempty
        create 640 mysql mysql
    postrotate
       # just if mysqld is really running
       if test -x /usr/bin/mysqladmin && \
          /usr/bin/mysqladmin --login-path=logrotate ping &>/dev/null
       then
          /usr/bin/mysqladmin --login-path=logrotate flush-logs
       fi
    endscript
}
```
### Configuring MySQL Authentication for Log Rotation

1.	Set up a secure login path to store the root user credentials:
`mysql_config_editor set --login-path=logrotate --user=root --host=localhost --password`
2.	Enter the root password when prompted. This command stores the credentials securely and avoids using plain text passwords in scripts.
3.	Verify the stored login paths: `mysql_config_editor print --all`
4.	Check the current permissions of the `~/.mylogin.cnf file`: `ls -l ~/.mylogin.cnf`
5.	If the permissions are not set to `-rw-------`, update them to ensure that only the file owner can read and write to it: `chmod 600 ~/.mylogin.cnf`
6. Repeat the previous steps for each additional MySQL DB VM to apply the same log rotation configuration.


## Logrotate Troubleshooting and Best Practices

### Resources for Logrotate
- Linux Shell Terminal: man logrotate
- Online Ressource: https://linux.die.net/man/8/logrotate

### Maintain Root Ownership for Logrotate Configuration Files
The logrotate configuration files should have permissions set to 644 and ownership set to root, which are the default settings. Additionally, the SELinux context type should be system_u:object_r:etc_t. When creating a new custom logrotate file, it is advisable to use cp -a <existing_logrotate_conf> <custom_new_logrotate_config> to ensure that the permissions, ownership, and SELinux settings are preserved.

### Verify Ownership, Permission, and SELinux settings of log files	
- Use the `ls -l` command to ensure that the ownership and permissions of the log files specified in the logrotate configuration file (e.g., etc/logrotate.d/<config_file>) match the settings provided in the logrotate configuration. For example: `create 640 mysql mysql`

- Use `ls-Z` to display the SELinux (Security-Enhanced Linux) security context of files and directories. 

### Understanding the ‘su’ Directive with Specific ‘create’ Settings
When logrotate is configured with `create 770 root adminops`, it sets new log files to 770 permissions, accessible only by the owner and group (root and adminops), and restricts access for others. Include the su directive (su root adminops) in the logrotate configuration to enforce this ownership.

### Run Manual Test	
Perform a manual test by executing the following command as root user:
`logrotate -vf /etc/logrotate.d/<config_file>`
Note: Ensure that the original log file contains log entries, as an empty log file will not be rotated.

### Debug Mode	
This mode is purely for verifying what logrotate would do under normal operational conditions. If you want to see logrotate performing the operations without making changes, run the following:  `logrotate --debug /etc/logrotate.d/<config_file>`

### Last Log Rotation Timestamps by Logrotate	
`cat /var/lib/logrotate/logrotate.status` 

### Compression Issue	
If the first rotated file is not being compressed, ensure that both the compress and copytruncate directives are declared in the logrotate configuration file.

### Verify Full File Paths	
Double-check that the full file paths of the log files are correctly declared in the logrotate configuration file. If uncertain, navigate to the directory containing the log files and run the command `pwd` to confirm.

### Check for Typos	
Review the logrotate configuration file for any typographical errors that may be causing issues.

### Curly Braces and Comments
Review the logrotate configuration file for any typographical errors that may be causing issues.




