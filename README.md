# AWS CloudWatch Monitoring for EC2 (Metrics, Logs & Alarms)

Monitoring an EC2-hosted Nginx web server with AWS CloudWatch — collecting logs, tracking key metrics, and firing email alerts on high resource usage or failures.

![Status](https://img.shields.io/badge/status-completed-brightgreen) ![AWS](https://img.shields.io/badge/AWS-CloudWatch-orange) ![Level](https://img.shields.io/badge/level-beginner--friendly-blue)

---

## Overview

As a DevOps Engineer, one of your core responsibilities is making sure problems are caught *before* users notice them. This project sets up complete observability for a web server running on EC2:

- **Logs** — centralize system and web server logs in CloudWatch
- **Metrics** — visualize CPU, network, and disk activity
- **Alarms** — get notified by email the moment something goes wrong

By the end, the server can be watched in real time, and you'll get an email the moment CPU spikes or the instance fails a health check.

---

## Architecture

```
                ┌─────────────────────┐
                │   EC2 Instance       │
                │   (Ubuntu + Nginx)   │
                │                       │
                │  CloudWatch Agent ────┼──► CloudWatch Logs
                │                       │      • syslog
                │                       │      • nginx access.log
                │                       │      • nginx error.log
                └──────────┬────────────┘
                           │
                           ▼
                 CloudWatch Metrics
             (CPU, Network In/Out, Disk I/O)
                           │
                           ▼
                 CloudWatch Alarms
              • High CPU (> 70%)
              • Status Check Failed
                           │
                           ▼
                     SNS Topic
                           │
                           ▼
                  📧 Email Notification
```

---

## Objective

Configure AWS CloudWatch to monitor an EC2 instance by collecting logs, tracking metrics, and creating alarms that notify you when the server experiences high resource utilization or application errors.

---

## Prerequisites

- An AWS account with permission to use EC2, CloudWatch, IAM, and SNS
- Basic familiarity with the Linux terminal
- An email address to receive alarm notifications

---

## Tools & Services Used

| Tool / Service | Purpose |
|---|---|
| Amazon EC2 | Hosts the Ubuntu server running Nginx |
| Nginx | Web server generating access/error logs |
| CloudWatch Agent | Ships logs from EC2 to CloudWatch Logs |
| CloudWatch Logs | Centralized log storage and viewing |
| CloudWatch Metrics | Visualizing CPU, network, and disk stats |
| CloudWatch Alarms | Threshold-based alerting |
| Amazon SNS | Sends email notifications when alarms trigger |
| stress-ng / curl | Simulate CPU load and web traffic for testing |

---

## Step-by-Step Walkthrough

### Task 1: Launch an EC2 Instance

1. Launch a new EC2 instance using an **Ubuntu** AMI (e.g., Ubuntu 22.04 LTS).
2. Choose an instance type such as `t2.micro` (Free Tier eligible).
3. Configure the Security Group to allow:
   - **SSH (port 22)** — from your IP
   - **HTTP (port 80)** — from anywhere (to view the web page)
4. Launch the instance and connect via SSH:
   ```bash
   ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
   ```
5. Install and start Nginx:
   ```bash
   sudo apt update
   sudo apt install nginx -y
   sudo systemctl enable nginx
   sudo systemctl start nginx
   ```
6. Verify the web page loads by visiting `http://<EC2_PUBLIC_IP>` in your browser.

*EC2 instance running + Nginx default page in browser*
`![EC2 and Nginx](screenshots/01-ec2-nginx.png)`

---

### Task 2: Configure CloudWatch Logs

1. Attach an IAM role to the EC2 instance with the **CloudWatchAgentServerPolicy** permission.
2. Install the CloudWatch Agent:
   ```bash
   wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
   sudo dpkg -i amazon-cloudwatch-agent.deb
   ```
3. Run the CloudWatch Agent configuration wizard instead of writing the config file by hand:
   ```bash
   sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
   ```
   Answer the prompts as follows:
   - **OS:** `linux`
   - **Which user to run agent:** `root` (default)
   - **Monitor host metrics:** `yes` (or `no` — Task 3 covers metrics separately)
   - **Do you want to monitor any log files:** `yes`
   - Add each log file when prompted, one at a time:
     - Log file path: `/var/log/syslog` → Log group name: `syslog`
     - Log file path: `/var/log/nginx/access.log` → Log group name: `nginx-access-log`
     - Log file path: `/var/log/nginx/error.log` → Log group name: `nginx-error-log`
   - **Do you want to turn on StatsD daemon?:** `no`
   - **Do you want the CloudWatch agent to also retrieve X-ray traces?L** `no`
   - When asked if you want to monitor more log files, choose **no** once all three are added.
   - At the end, choose to **store the config directly** (the wizard saves it to `/opt/aws/amazon-cloudwatch-agent/bin/config.json.` automatically).

4. Start the agent using the config file the wizard just created:
   ```bash
   sudo systemctl start amazon-cloudwatch-agent
   sudo systemctl status amazon-cloudwatch-agent
   ```

*CloudWatch agent is started*
`![Log Groups](screenshots/02-cloudwatch-agent.png)`

5. In the AWS Console, go to **CloudWatch → Log management** and confirm the three log groups have been created.

*CloudWatch log groups list*
`![Log Groups](screenshots/02-log-groups.png)`

6. Refresh the Nginx web page a few times, then check that new log entries appear in real time.

*Live log entries in a log stream*
`![Log Stream](screenshots/03-log-stream.png)`

---

### Task 3: Monitor CloudWatch Metrics

1. Go to **CloudWatch → Classic Metrics → Under Browse() → AWS namespaces → EC2 → Per-Instance Metrics**.
2. Select your instance and view the following metrics:
   - CPU Utilization
   - Network In
   - Network Out
   - Disk Read Operations or EBSReadOps
   - Disk Write Operations or EBSWriteOps
3. Add the metrics to a graph to visualize trends over time.

*CloudWatch metrics dashboard with graphs*
`![Metrics Dashboard](screenshots/04-metrics-dashboard.png)`

---

### Task 4: Create CloudWatch Alarms

#### Alarm 1: High CPU Usage

1. Go to **CloudWatch → Alarms → Create alarm**.
2. Select the **CPUUtilization** metric for your instance.
3. Set:
   - **Threshold:** Greater than 70%
   - **Evaluation Period:** 2 consecutive periods
4. Create an **SNS topic** (e.g., `ec2-alerts`) and subscribe your email address.
5. Confirm the subscription from the confirmation email AWS sends you.
6. Attach the SNS topic as the alarm action.

*High CPU alarm configuration*
`![CPU Alarm Config](screenshots/05-cpu-alarm-config.png)`

#### Alarm 2: EC2 Status Check Failure

1. Create another alarm using the **StatusCheckFailed** metric.
2. Set the threshold to **greater than 0**.
3. Use the same SNS topic to send notifications.

*Status check alarm configuration*
`![Status Check Alarm Config](screenshots/06-status-check-alarm-config.png)`

---

### Task 5: Test the Alarms

**Generate CPU load** to trigger the High CPU alarm:
```bash
sudo apt install stress-ng -y
stress-ng --cpu 2 --timeout 300s
```

**Generate web traffic** to see network metrics move:
```bash
while true; do curl http://<EC2_PUBLIC_IP>; done
```

**Observe:**
- The CPU alarm transitions from **OK → ALARM** during the stress test.
- Network In/Out metrics rise with repeated HTTP requests.
- An email notification arrives via SNS when the alarm triggers.

*Alarm in ALARM state*
`![Alarm Triggered](screenshots/07-alarm-triggered.png)`

*SNS email notification received*
`![SNS Email](screenshots/08-sns-email.png)`

---

### Task 6: Verify Monitoring

Final checklist to confirm everything works end-to-end:

- [ ] CloudWatch Logs contain Nginx access and error logs
- [ ] CloudWatch Metrics display CPU and network activity
- [ ] CloudWatch Alarms transition correctly between OK and ALARM
- [ ] SNS email notifications are received when alarms are triggered

*Alarm history showing OK ↔ ALARM transitions*
`![Alarm History](screenshots/09-alarm-history.png)`

---

## Key Learnings

- How to install and configure the CloudWatch Agent to ship custom log files from an EC2 instance
- The difference between default EC2 metrics (CPU, network, disk) available without any agent, versus custom logs which require the agent
- How to create threshold-based CloudWatch Alarms and wire them to an SNS topic for email alerts
- How to simulate real failure/load conditions to validate that monitoring and alerting actually work

---

## Cleanup

To avoid ongoing charges, remember to:
1. Terminate the EC2 instance
2. Delete the CloudWatch log groups (optional — logs may incur storage cost)
3. Delete the CloudWatch alarms
4. Delete the SNS topic and subscription

---

## Author

**Sinsha C**
Senior DevOps Engineer | Cloud & Infrastructure Specialist
[GitHub](https://github.com/sinsha-c) • [LinkedIn](https://linkedin.com/in/sinshac/)