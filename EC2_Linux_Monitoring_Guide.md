
# EC2 & Linux Monitoring – Practical Guide

**Topics:** CPU, Memory, Disk, Inodes, Disk I/O, Network, Apache HTTPD, CloudWatch Agent, Logs, Alarms and Troubleshooting.

---

## 1. CPU Monitoring

### What is CPU utilization?

CPU utilization tells us how much of the EC2 instance's processing capacity is being used.

- 10% CPU – Low CPU usage.
- 50% CPU – Half of the available CPU capacity is being used.
- 100% CPU – All available CPU capacity is being used.

### Important CPU metrics

| Metric | Meaning |
|---|---|
| `CPUUtilization` | Overall EC2 CPU utilization |
| `cpu_usage_user` | CPU used by applications running in user space |
| `cpu_usage_system` | CPU consumed by the Linux kernel |
| `cpu_usage_idle` | CPU time when the processor is idle |
| `cpu_usage_iowait` | CPU time spent waiting for I/O operations |

**Important difference:**

- `CPUUtilization` is an EC2-level metric available through AWS.
- `cpu_usage_user`, `cpu_usage_system`, etc. are detailed metrics collected by the CloudWatch Agent.

### User CPU vs System CPU

**User CPU:** CPU consumed by applications executing their own code.

Example: Apache processing an HTTP request or Python executing a script.

**System CPU:** CPU consumed by the Linux kernel while performing operations on behalf of applications.

Example: Processing network packets, managing memory and handling system calls.

### Practical: Generate CPU load

Install `stress-ng` on Amazon Linux:

```bash
sudo dnf install stress-ng -y
```

Generate CPU load:

```bash
stress-ng --cpu 2 --timeout 120s
```

This creates two CPU-intensive worker processes for 120 seconds.

Monitor CPU:

```bash
top
```

Or:

```bash
mpstat 1
```

### Why does Apache still respond when CPU reaches 100%?

Even if CPU utilization reaches 100%, Apache may continue responding because:

1. Linux uses CPU scheduling to distribute processing time among processes.
2. Apache gets opportunities to execute when scheduled.
3. Requests may be lightweight and require very little CPU.
4. Some requests may be served from memory or the operating system's cache.

However, under sustained CPU saturation, response time can increase and requests may eventually time out.

**Remember:** 100% CPU does not automatically mean an application is down.

---

## 2. Memory Monitoring

### What is memory utilization?

Memory utilization indicates how much RAM is being used by the operating system and applications.

Example:

An EC2 instance has 2 GB RAM.

- Used: 1.5 GB
- Available: 500 MB
- Memory utilization: approximately 75%

### Important memory metrics

| Metric | Meaning |
|---|---|
| `mem_total` | Total RAM |
| `mem_used` | Memory currently used |
| `mem_available` | Memory estimated to be available for new applications |
| `mem_free` | Completely unused memory |
| `mem_used_percent` | Percentage of memory used |
| `swap_used` | Swap space currently used |

### Why is free memory low in Linux?

Linux uses available RAM for caching frequently accessed files and data.

This improves performance.

Therefore:

**Low free memory does not necessarily mean memory shortage.**

The more useful metric to observe is `MemAvailable`.

### Practical commands

Check memory:

```bash
free -h
```

Check memory usage by processes:

```bash
top
```

Check memory details:

```bash
cat /proc/meminfo
```

### What happens when memory is exhausted?

When Linux experiences severe memory pressure, it may invoke the **OOM Killer (Out Of Memory Killer)**.

Its purpose is to terminate selected processes to recover memory and keep the system running.

For example:

1. Apache consumes increasing memory.
2. Available RAM becomes critically low.
3. The kernel cannot satisfy memory allocation requests.
4. The OOM Killer selects a process.
5. That process may be terminated.

### What is SIGKILL?

SIGKILL is a signal that immediately terminates a process.

```bash
kill -9 <PID>
```

Important points:

- A process cannot catch or ignore SIGKILL.
- It cannot perform normal cleanup after receiving SIGKILL.
- The OOM Killer can use SIGKILL to terminate a process.

**Important:** The process killed by the OOM Killer is not necessarily the process that originally caused memory pressure.

### Check OOM events

```bash
sudo dmesg -T | grep -i -E 'oom|killed process'
```

Or:

```bash
sudo journalctl -k | grep -i oom
```

---

## 3. Disk Space Monitoring

### What is disk utilization?

Disk utilization tells us how much storage capacity has been consumed.

Example:

A 20 GB filesystem has 15 GB occupied.

Disk utilization = 75%.

### Important commands

Check filesystem usage:

```bash
df -h
```

Check directory size:

```bash
du -sh /var/log
```

Check the largest directories:

```bash
sudo du -ah /var | sort -rh | head -10
```

### Difference between `df` and `du`

| Command | Purpose |
|---|---|
| `df -h` | Shows filesystem-level disk usage |
| `du -sh` | Shows how much space a directory consumes |

Example:

`df -h` shows that the root filesystem is 80% full.

`sudo du -sh /var/log` shows that `/var/log` occupies 5 GB.

This helps identify where disk space is being consumed.

---

## 4. Inodes – An Important Linux Concept

### What is an inode?

An inode is a data structure in a Linux filesystem that stores information about a file.

It contains information such as:

- File permissions
- File ownership
- File size
- Timestamps
- Disk block locations

**Important:** The filename itself is not stored in the inode. The directory maintains the mapping between filenames and inode numbers.

### What happens when inodes are exhausted?

Linux needs an available inode to create a new file.

Suppose:

- Disk capacity: 100 GB
- Used disk space: 20 GB
- Available disk space: 80 GB
- Available inodes: 0

You may still be unable to create new files, even though plenty of disk space is available.

### Practical commands

Check inode utilization:

```bash
df -i
```

Find inode number:

```bash
ls -i filename
```

View file metadata:

```bash
stat filename
```

### Disk space vs Inode utilization

| Disk space | Inodes |
|---|---|
| Measures storage capacity | Represents filesystem objects |
| Consumed by file contents and metadata | Generally one inode per filesystem object |
| Checked using `df -h` | Checked using `df -i` |
| Can become full | Can be exhausted independently |

**Real-world example:** An application continuously creates millions of small log files. Disk space may remain available, but inode exhaustion can prevent further file creation.

---

## 5. Disk I/O Monitoring

### What is Disk I/O?

Disk I/O means Input/Output operations performed between the operating system and storage devices.

- Read: Reading data from disk.
- Write: Writing data to disk.

For EC2, the underlying storage could be Amazon EBS.

### Important disk I/O metrics

| Metric | Meaning |
|---|---|
| ReadBytes | Number of bytes read |
| WriteBytes | Number of bytes written |
| ReadOps | Number of read operations |
| WriteOps | Number of write operations |
| await | Average time taken for I/O requests |
| aqu-sz | Average number of outstanding I/O requests |
| %util | Percentage of time the device is busy handling I/O |

### Practical commands

Install sysstat:

```bash
sudo dnf install sysstat -y
```

Monitor disk I/O:

```bash
iostat -xz 1
```

This refreshes the disk I/O statistics every second.

### What should we observe?

- `r/s` – Read operations per second.
- `w/s` – Write operations per second.
- `rMB/s` – Read throughput.
- `wMB/s` – Write throughput.
- `await` – Average I/O request completion time.
- `%util` – Device busy time.

### Important understanding

A disk can have plenty of free space but still experience poor performance.

Example:

- Disk capacity: 100 GB
- Used: 20 GB
- Disk utilization: 20%
- Disk I/O latency: Very high

The issue is not storage capacity. It could be storage performance limitations or heavy I/O activity.

For EBS, also consider volume IOPS and throughput limits.

---

## 6. Network Monitoring

Network monitoring helps us understand how much traffic an EC2 instance is receiving and sending.

### Important metrics

| Metric | Meaning |
|---|---|
| NetworkIn | Bytes received by the instance |
| NetworkOut | Bytes sent by the instance |
| net_bytes_recv | Bytes received, collected by CloudWatch Agent |
| net_bytes_sent | Bytes sent, collected by CloudWatch Agent |
| NetworkPacketsIn | Incoming packets |
| NetworkPacketsOut | Outgoing packets |

### Practical commands

View network interfaces:

```bash
ip addr
```

Check network connections:

```bash
ss -ant
```

Check listening ports:

```bash
sudo ss -lntp
```

Check network statistics:

```bash
ip -s link
```

Check overall TCP statistics:

```bash
ss -s
```

### Example

Suppose your EC2 instance hosts an Apache web server.

- NetworkIn increases when users send HTTP requests.
- NetworkOut increases when Apache sends responses.

If NetworkIn is increasing but NetworkOut is very low, investigate whether the application is receiving requests but failing to respond.

This is only a troubleshooting clue, not proof of an application failure.

---

## 7. Process and Application Monitoring

Monitoring CPU, memory and disk is not enough. We also need to know whether important applications are running.

For example, Apache HTTPD.

### Useful commands

Check Apache service:

```bash
sudo systemctl status httpd
```

Check Apache processes:

```bash
pgrep -a httpd
```

Check whether Apache is listening on port 80:

```bash
sudo ss -lntp | grep ':80'
```

Test the application locally:

```bash
curl -I http://localhost/
```

Check Apache logs:

```bash
sudo tail -f /var/log/httpd/access_log
```

```bash
sudo tail -f /var/log/httpd/error_log
```

### Important understanding

There are different levels of application health:

| Check | What it confirms |
|---|---|
| Process exists | Application process is running |
| Port is listening | Application is accepting connections on that port |
| HTTP health check | Application is responding to HTTP requests |
| Application-level health check | Application's important functionality is working |

**Example:** Apache may be running and port 80 may be listening, but the application could still return HTTP 500 errors.

Therefore, a proper health check is more useful than simply checking whether the process exists.

---

## 8. CloudWatch Monitoring

Amazon CloudWatch is an AWS monitoring and observability service.

It helps us collect, monitor and analyze metrics, logs and events from AWS resources and applications.

### CloudWatch metrics

A metric is a numerical measurement collected over time.

Examples:

- CPU utilization: 75%
- Memory utilization: 80%
- Disk utilization: 90%
- Network traffic: 10 MB/s

### CloudWatch namespaces

A namespace is a container used to organize metrics.

| Namespace | Purpose |
|---|---|
| AWS/EC2 | Native EC2 metrics |
| CWAgent | Metrics collected by CloudWatch Agent |
| Custom namespace | Application-specific metrics |

### Important distinction

EC2 provides basic CPU, network and certain storage activity metrics by default.

However, memory utilization and filesystem-level disk utilization are not available as standard EC2 metrics by default.

For these, we generally install the CloudWatch Agent.

---

## 9. CloudWatch Agent

The CloudWatch Agent is software installed on an EC2 instance to collect additional system metrics and logs.

### Why do we need it?

By default, EC2 does not provide all the operating-system-level metrics we need.

The CloudWatch Agent helps collect:

- Memory utilization
- Disk utilization
- Disk I/O
- Swap utilization
- Process statistics
- Apache access logs
- Apache error logs

### Example CloudWatch Agent metrics

| Metric | Purpose |
|---|---|
| mem_used_percent | Memory utilization |
| mem_available | Available memory |
| disk_used_percent | Filesystem utilization |
| disk_free | Available filesystem space |
| diskio_read_bytes | Disk read bytes |
| diskio_write_bytes | Disk write bytes |
| cpu_usage_user | User CPU utilization |
| cpu_usage_system | System CPU utilization |
| swap_used_percent | Swap utilization |
| procstat_lookup_pid_count | Number of matching processes |

### Apache log paths on Amazon Linux

```text
/var/log/httpd/access_log
/var/log/httpd/error_log
```

These logs can be collected by CloudWatch Agent and sent to CloudWatch Logs.

---

## 10. CloudWatch Logs – Log Group vs Log Stream

### What is a Log Group?

A Log Group is a logical container used to organize application logs.

Example:

```text
/ec2/apache
```

You can keep all Apache-related logs under this Log Group.

### What is a Log Stream?

A Log Stream is a sequence of log events from a particular source.

For example:

```text
Log Group: /ec2/apache

    Log Stream: i-0123456789-access
    Log Stream: i-0123456789-error
```

Here:

- The Log Group organizes Apache logs.
- The access stream contains HTTP access logs.
- The error stream contains Apache error logs.

### Simple real-world analogy

Think of a Log Group as a folder.

Inside that folder, Log Streams are individual files containing log events from different sources.

### What is a Log Event?

A Log Event is an individual log entry with a timestamp and message.

Example:

```text
223.185.45.210 - - [22/Nov/2026:10:35:19 +0000] "GET / HTTP/1.1" 200 1024
```

This is one Apache access log event.

### Why separate access and error logs?

**Access logs** help us understand:

- Who accessed the server?
- Which URL was requested?
- What HTTP status code was returned?
- How much data was returned?

**Error logs** help us understand:

- Why a request failed.
- Configuration problems.
- Permission issues.
- Application and server errors.

---

## 11. CloudWatch Alarms

A CloudWatch Alarm monitors a metric and changes state when a configured condition is met.

### Example: CPU Alarm

```text
Metric: CPUUtilization
Condition: Greater than 80%
Period: 5 minutes
Evaluation periods: 2
```

If CPU utilization stays above 80% for two consecutive evaluation periods, the alarm can enter the ALARM state.

### Recommended alarms for EC2

| Metric | Example threshold | Why monitor it? |
|---|---:|---|
| CPUUtilization | Above 80% | Detect high CPU usage |
| mem_used_percent | Above 85% | Detect memory pressure |
| disk_used_percent | Above 85% | Detect disk capacity issues |
| Swap utilization | Above 50% | Investigate memory pressure |
| Process count | Below 1 | Detect application process failure |
| HTTP health check | Failed | Detect application unavailability |
| Disk I/O latency | Unusually high | Detect storage performance problems |

These are starting examples, not universal production thresholds. Tune them based on normal workload patterns.

---

## 12. What is Anomaly Detection?

Traditional alarms use fixed thresholds.

Example:

```text
CPU utilization > 80%
```

But what if your application normally runs at 10% CPU and suddenly reaches 50%?

A traditional alarm might not trigger because CPU is still below 80%.

CloudWatch Anomaly Detection uses historical metric patterns to build a model of expected behavior.

It creates an expected range, often called an anomaly detection band.

### Example

Suppose your server normally experiences:

- Morning CPU: 20%
- Afternoon CPU: 40%
- Night CPU: 10%

One day, CPU suddenly reaches 75% during the night.

Even though CPU is below a fixed 80% alarm threshold, it may be outside the expected range.

Anomaly Detection can identify this unusual behavior.

### Traditional alarm vs Anomaly Detection

| Traditional alarm | Anomaly Detection |
|---|---|
| Uses a fixed threshold | Uses historical patterns |
| Example: CPU above 80% | Detects unusual CPU behavior |
| Easy to configure | Learns expected metric behavior |
| May miss unusual but below-threshold spikes | Can detect unexpected deviations |

---

## 13. Common EC2 Failure Scenarios

| Problem | Possible symptom | What to check |
|---|---|---|
| CPU saturation | Slow responses | `top`, `mpstat` |
| Memory exhaustion | Processes killed | `free -h`, `dmesg` |
| Disk full | Applications cannot write files | `df -h` |
| Inode exhaustion | Cannot create files | `df -i` |
| High disk latency | Slow application operations | `iostat -xz 1` |
| Apache stopped | Website unavailable | `systemctl status httpd` |
| Apache not listening | Connection refused | `ss -lntp` |
| Network issue | Connection timeout | `ss`, `ip -s link` |
| High error rate | HTTP 4xx/5xx responses | Apache access/error logs |

---

## 14. Practical Troubleshooting: Apache Website Is Not Working

Suppose someone reports that your website hosted on EC2 is not accessible.

Follow this sequence.

### Step 1: Check HTTP response

```bash
curl -I http://localhost/
```

If this fails, investigate Apache or the local application first.

### Step 2: Check Apache service

```bash
sudo systemctl status httpd
```

If stopped:

```bash
sudo systemctl start httpd
```

### Step 3: Check Apache process

```bash
pgrep -a httpd
```

### Step 4: Check listening port

```bash
sudo ss -lntp | grep ':80'
```

### Step 5: Check Apache error logs

```bash
sudo tail -50 /var/log/httpd/error_log
```

### Step 6: Check system logs

```bash
sudo journalctl -u httpd -n 50
```

### Step 7: Check memory

```bash
free -h
```

Check whether the OOM Killer terminated any processes:

```bash
sudo dmesg -T | grep -i -E 'oom|killed process'
```

### Step 8: Check disk space and inodes

```bash
df -h
```

```bash
df -i
```

### Step 9: Check CPU and disk I/O

```bash
top
```

```bash
iostat -xz 1
```

### Step 10: Check AWS networking

If the application works locally but cannot be accessed externally, verify:

- EC2 Security Group inbound rules.
- Network ACL rules.
- Route table.
- Internet Gateway.
- Public IP address.
- Apache listening address and port.

**Troubleshooting principle:** Start from the application and move outward toward the network.

---

## 15. Interview Questions and Answers

### Q1. Why doesn't AWS provide memory utilization by default for EC2?

EC2 basic monitoring does not collect operating-system-level memory statistics. We install the CloudWatch Agent to collect memory metrics from inside the instance.

### Q2. What is the difference between CPU utilization and memory utilization?

CPU utilization measures processor usage. Memory utilization measures RAM consumption.

High CPU indicates heavy processing activity. High memory utilization indicates that applications and the OS are consuming available RAM.

### Q3. If CPU utilization reaches 100%, will the application stop?

Not necessarily. Linux schedules CPU time among processes. The application may continue responding, but response times can increase and requests may eventually time out.

### Q4. What is the difference between free memory and available memory?

Free memory is completely unused RAM. Available memory estimates how much memory can be used by new applications without requiring significant swapping.

### Q5. What is the OOM Killer?

The OOM Killer is a Linux kernel mechanism that terminates selected processes when the system experiences severe memory exhaustion.

### Q6. Can a process ignore SIGKILL?

No. SIGKILL cannot be caught, blocked or ignored.

### Q7. Can a disk be full even when free space is available?

Yes. Inode exhaustion can prevent new files from being created even when disk storage capacity remains available.

### Q8. What is the difference between disk utilization and disk I/O utilization?

Disk utilization generally refers to consumed storage capacity. Disk I/O metrics describe read/write activity and storage performance.

### Q9. What is the difference between `df` and `du`?

`df` reports filesystem usage. `du` reports how much space files and directories consume.

### Q10. What is the difference between a Log Group and a Log Stream?

A Log Group is a logical container for logs. A Log Stream contains log events from a particular source.

### Q11. Why should we collect Apache access and error logs?

Access logs help analyze incoming requests and HTTP response codes. Error logs help troubleshoot server and configuration problems.

### Q12. What is CloudWatch Anomaly Detection?

It uses historical metric patterns to estimate normal behavior and identify unusual deviations from that behavior.

### Q13. If Apache is running, does that mean the website is healthy?

No. The process can be running while the application returns errors. We should also check the listening port and HTTP response.

### Q14. How would you monitor an EC2 instance in production?

I would monitor CPU, memory, disk utilization, inode usage, disk I/O, network traffic and application health. I would use CloudWatch Agent for additional system metrics and log collection, CloudWatch Logs for centralized log analysis, and CloudWatch Alarms for proactive alerting.

---

## 16. Hands-on Practice Exercises

### Exercise 1: CPU monitoring

**Level:** Beginner

1. Open two SSH sessions to your EC2 instance.
2. In the first session, run:

```bash
top
```

3. In the second session, generate CPU load:

```bash
stress-ng --cpu 2 --timeout 120s
```

4. Observe CPU utilization.
5. Test Apache while CPU is high:

```bash
curl -w "\nTime: %{time_total}s\n" -o /dev/null -s http://localhost/
```

**Understand:** Does Apache stop responding at 100% CPU? Does response time change?

### Exercise 2: Memory monitoring

**Level:** Beginner

1. Run:

```bash
free -h
```

2. Check memory-consuming processes:

```bash
top
```

3. Observe the difference between `free` and `available`.

**Understand:** Why can Linux use memory for caching?

### Exercise 3: Disk and inode monitoring

**Level:** Intermediate

1. Run:

```bash
df -h
df -i
```

2. Create a temporary directory with many small files:

```bash
mkdir -p /tmp/inode-test
for i in $(seq 1 10000); do touch /tmp/inode-test/file-$i; done
```

3. Run `df -i` again and observe inode usage.
4. Clean up:

```bash
rm -rf /tmp/inode-test
```

**Understand:** Why does creating many small files consume inodes?

### Exercise 4: Apache log monitoring

**Level:** Beginner

1. Open a terminal and monitor access logs:

```bash
sudo tail -f /var/log/httpd/access_log
```

2. Open another terminal and monitor error logs:

```bash
sudo tail -f /var/log/httpd/error_log
```

3. Generate requests:

```bash
curl http://localhost/
```

**Understand:** Which log records a successful request? Which log records server-side errors?

### Exercise 5: CloudWatch monitoring

**Level:** Intermediate

1. Install the CloudWatch Agent.
2. Configure memory, disk and CPU metrics.
3. Configure Apache access and error log collection.
4. Start the agent.
5. Verify metrics under the `CWAgent` namespace.
6. Verify logs under CloudWatch Logs.
7. Create a CPU utilization alarm.

**Understand:** Which metrics are native EC2 metrics, and which are collected by the agent?

---

## 17. Quick Command Reference

| Command | Purpose |
|---|---|
| `top` | Live process and CPU/memory monitoring |
| `free -h` | Memory utilization |
| `df -h` | Filesystem disk usage |
| `df -i` | Inode utilization |
| `du -sh /path` | Directory size |
| `iostat -xz 1` | Disk I/O statistics |
| `mpstat 1` | CPU statistics |
| `vmstat 1` | CPU, memory and system activity |
| `pgrep -a httpd` | Find Apache processes |
| `systemctl status httpd` | Check Apache service |
| `journalctl -u httpd` | Apache service logs |
| `ss -lntp` | Listening TCP ports |
| `ss -ant` | TCP connections |
| `ip -s link` | Network interface statistics |
| `dmesg -T` | Kernel messages |
| `curl -I http://localhost/` | Test HTTP response |
| `tail -f /var/log/httpd/access_log` | Live Apache access logs |
| `tail -f /var/log/httpd/error_log` | Live Apache error logs |

---

## 18. Interview Explanation Framework

Whenever an interviewer asks about a Linux or AWS concept, use this structure:

1. **What:** Define the concept in simple words.
2. **Why:** Explain why it is required.
3. **Example:** Give a real-world example.
4. **How to check:** Mention the command or monitoring metric.
5. **What next:** Explain how you would troubleshoot it.

### Example: How do you troubleshoot high memory utilization?

**What:** First, I understand that memory utilization indicates how much RAM is being consumed.

**Why:** High memory usage can affect application performance and may eventually cause the Linux OOM Killer to terminate processes.

**Example:** If Apache starts consuming excessive memory, available RAM can decrease significantly.

**How to check:**

```bash
free -h
top
```

**What next:** I identify the process consuming memory, check system logs for OOM events, and investigate whether the application has a memory leak or an unexpected workload.

---

## Final Learning Tip

Don't just memorize the commands. For every metric, understand three things:

1. What does it measure?
2. What happens when it becomes abnormal?
3. How would you troubleshoot it?

That is what will help you explain these concepts confidently in DevOps interviews.
