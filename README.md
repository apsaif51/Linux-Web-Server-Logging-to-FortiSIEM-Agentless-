# Linux-Web-Server-Logging-to-FortiSIEM-Agentless
## Overview
This repository demonstrates how to forward Linux web server logs to FortiSIEM using rsyslog, without installing the FortiSIEM Agent.  The deployment was implemented on legacy CentOS servers where installing the FortiSIEM Agent introduced compatibility issues.  Instead of relying on the agent, Apache access and error logs are collected locally and forwarded securely to FortiSIEM through **Syslog**.

## Environment

| Component         | Version          |
| ----------------- | ---------------- |
| Linux             | CentOS 8 (Legacy)|
| Web Server        | Apache HTTPD     |
| SIEM              | FortiSIEM        |
| Log Forwarder     | rsyslog          |
| Collection Method | Agentless Syslog |

```text
  Apache
      │
      ▼
Access Log
Error Log
Virtual Host Logs (other vhosts)

      │
      ▼
  rsyslog

      │
      ▼
  UDP/TCP 514
      │

      ▼

  FortiSIEM Supervisor

```
Example:
/var/log/httpd/access_log
/var/log/httpd/error_log
/var/log/httpd/other_vhosts_access_log


## Why Agentless?

During deployment, the FortiSIEM Agent could not be reliably installed because:
Legacy CentOS version
End-of-Life operating system
Package dependency issues
Log writing conflicts

To avoid introducing instability into production systems, rsyslog was selected as the log forwarding mechanism.

## Problem Encountered

```text
 Apache logs(facility local6 choosed)
    ↓
  rsyslog
    ↓
  messages (the size of messages increases rapidly)
    ↓
  rsyslog reads messages again
    ↓
  Forwarded again (duplicated logs)
    ↓
  Loop
```

This created duplicate events and unnecessary log growth.


### Root Cause

The forwarded logs were being written back into **/var/log/messages.**

Since rsyslog was already monitoring that file, it created a forwarding loop.


## Solution

The solution was to exclude the logging facility from the default rules.

in **/etc/rsyslog.conf **:

`*.info;mail.none;authpriv.none;cron.none;local6.none /var/log/messages`

Using **local6.none** prevents logs tagged with the **local6 facility** from being written into **/var/log/messages.**

## Workflow
```text
  Apache
    ↓
  access.log
    ↓
  imfile
    ↓
  local6
    ↓
  rsyslog
    ↓
  FortiSIEM Supervisor
```

No duplicate logs.

No forwarding loop.

Cleaner local logging.

## Apache Configuration

Each VirtualHost (app) has its own logs.

### Example:

**Application 1**

```text
Access:
/var/log/httpd/app1_access.log

Error:
/var/log/httpd/app1_error.log
```
**Application 2**
```text

Access:
/var/log/httpd/app2_access.log

Error:
/var/log/httpd/app2_error.log
```
Every file is monitored independently by rsyslog.
  
