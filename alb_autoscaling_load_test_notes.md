# ApacheBench Load Testing Through an AWS Application Load Balancer

This note captures the load test performed against the PHP endpoint
`/infra.php` through an AWS Application Load Balancer (ALB), explains
the results, and lists the next checks for validating EC2 Auto Scaling.

------------------------------------------------------------------------

# Version 1: Detailed Notes

## 1. Test objective

Generate concurrent HTTP traffic through the ALB to observe how the
application and EC2 Auto Scaling environment behave under load.

**Endpoint tested**

``` text
http://app-alb-905324712.ap-south-1.elb.amazonaws.com/infra.php
```

**Test configuration**

  Setting                                                     Value
  ---------------------------- ------------------------------------
  Tool                                           ApacheBench (`ab`)
  Concurrency level                                             500
  Completed requests                                         75,019
  Test duration                                     454.154 seconds
  Test termination               Manually interrupted with `Ctrl+C`
  Response-length validation         Variable length allowed (`-l`)

Example command:

``` bash
ab -l -n 100000 -c 500 http://YOUR-ALB-DNS-NAME/infra.php
```

-   `-l`: Accept variable response lengths. Useful when a dynamic
    endpoint returns slightly different-sized responses.
-   `-n 100000`: Number of requests to attempt.
-   `-c 500`: Number of concurrent requests.

The command above is an example of the test configuration; the recorded
run was stopped manually after 75,019 requests.

## 2. Benchmark results

  Metric                                                          Result
  -------------------------------------------------- -------------------
  Concurrency Level                                                  500
  Time taken for tests                                   454.154 seconds
  Complete requests                                               75,019
  Failed requests                                                      0
  Non-2xx responses                                                   67
  Total transferred                                    591,100,457 bytes
  HTML transferred                                     577,673,999 bytes
  Requests per second                                             165.18
  Mean time per request                                     3,026.929 ms
  Mean time per request across concurrent requests              6.054 ms
  Transfer rate                                          1,271.04 KB/sec

### Response-time distribution

  Percentile         Response time
  ---------------- ---------------
  50% (median)            2,844 ms
  66%                     3,657 ms
  75%                     4,215 ms
  80%                     4,540 ms
  90%                     5,491 ms
  95%                     7,509 ms
  98%                     8,372 ms
  99%                    10,136 ms
  100% (maximum)         14,009 ms

## 3. Understanding the important metrics

### Requests per second: 165.18

During this run, ApacheBench recorded an average throughput of
approximately 165 completed requests per second.

This is the observed throughput for this particular test setup. It is
not automatically the maximum capacity of the ALB or the application.
The result depends on the client machine, network, application, EC2
instance types, number of targets, and workload.

### Concurrency: 500

ApacheBench maintained up to 500 concurrent requests.

Concurrency is not the same as requests per second. Concurrency
describes how many requests can be in progress at the same time;
requests per second describes throughput over time.

### Mean response time: approximately 3 seconds

The mean time per request was approximately 3,027 ms. This is the
average request time reported by ApacheBench for the run.

ApacheBench also reports `Time per request` across all concurrent
requests (approximately 6.054 ms). This second figure accounts for the
concurrency level and should not be interpreted as the typical end-user
response time.

### 95th percentile: approximately 7.5 seconds

The 95th percentile means 95% of the completed requests finished in
7.509 seconds or less. The slowest 5% took longer.

Percentiles help reveal slow requests that an average can hide.

### Failed requests: 0

ApacheBench did not classify any requests as failed under its request
checks.

However, this does **not** mean every response was successful: the
output also reports 67 non-2xx responses.

### Non-2xx responses: 67

These are responses whose HTTP status codes were outside the 200--299
range. They may include redirects (3xx), client errors (4xx), or server
errors (5xx).

The benchmark summary does not identify which status codes occurred.
Capture and inspect the actual HTTP status codes before concluding that
these were application or server errors.

## 4. Why use `-l`?

Earlier tests showed `Failed requests (Length)` because the PHP endpoint
returned response bodies with slightly different sizes.

ApacheBench normally compares response body lengths against the
reference document length. The `-l` option tells it to accept variable
response lengths.

This changes ApacheBench's response-length validation; it does not fix
or change the application itself.

## 5. Important limitation of this run

The test was manually stopped with `Ctrl+C` after 75,019 completed
requests. Therefore, it is a partial run rather than a completed
fixed-request benchmark.

Also, this single result does not prove that the ALB or Auto Scaling
Group scaled successfully. To validate scaling, correlate the load-test
time with CloudWatch metrics and the Auto Scaling activity history.

## 6. What to monitor in AWS

During a repeatable load test, monitor the following metrics:

  -----------------------------------------------------------------------------
  AWS component           Metric                        Why it matters
  ----------------------- ----------------------------- -----------------------
  Application Load        `RequestCount`                Total requests received
  Balancer                                              by the ALB

  Target group            `RequestCountPerTarget`       Average request volume
                                                        per registered target;
                                                        useful for
                                                        request-based scaling

  Target group            `TargetResponseTime`          Time taken for targets
                                                        to respond

  Load Balancer / Target  `HTTPCode_Target_5XX_Count`   Server-side errors
  group                                                 returned by targets

  EC2                     `CPUUtilization`              CPU pressure on
                                                        instances

  Auto Scaling Group      `GroupInServiceInstances`     Number of instances
                                                        currently in service

  Auto Scaling Group      Scaling activities            Whether scale-out or
                                                        scale-in actions
                                                        occurred
  -----------------------------------------------------------------------------

For a target-tracking policy based on ALB request count per target,
check the target group's `RequestCountPerTarget` metric and the scaling
policy's configured target value.

## 7. Recommended next experiment

1.  Record the current number of healthy ALB targets and EC2 instances
    in the Auto Scaling Group.
2.  Start a load test from a separate machine, not from one of the
    application instances.
3.  Monitor `RequestCountPerTarget`, `TargetResponseTime`, EC2 CPU, and
    Auto Scaling activities.
4.  Record when new instances launch and when they become healthy
    targets.
5.  Repeat the test with a lower concurrency (for example, 50 or 100)
    and compare the results.
6.  Check the 67 non-2xx responses by identifying their HTTP status
    codes.

Useful command for a controlled repeat test:

``` bash
ab -l -n 100000 -c 500 http://YOUR-ALB-DNS-NAME/infra.php
```

Use a test environment and monitor resource consumption. High
concurrency can put significant pressure on the load generator and the
application.

## 8. Questions to answer after the experiment

-   Did the Auto Scaling Group launch additional EC2 instances?
-   How many targets were healthy before and after the test?
-   Did `RequestCountPerTarget` cross the configured target-tracking
    threshold?
-   Did target response time improve after new instances became healthy?
-   What HTTP status codes account for the 67 non-2xx responses?
-   Was the load generator itself able to sustain the required traffic?

------------------------------------------------------------------------

# Version 2: Quick Recap (Interview Revision)

## ApacheBench + ALB load test

**Objective:** Generate concurrent traffic against an application hosted
on EC2 instances behind an ALB and observe performance and scaling
behaviour.

**Example command:**

``` bash
ab -l -n 100000 -c 500 http://YOUR-ALB-DNS-NAME/infra.php
```

  Option   Meaning
  -------- --------------------------------------
  `-l`     Allow variable response-body lengths
  `-n`     Total number of requests
  `-c`     Number of concurrent requests

### Recorded results

-   Concurrency: **500**
-   Completed requests: **75,019**
-   Duration: **454.154 seconds**
-   Throughput: **165.18 requests/sec**
-   Mean request time: **3,026.929 ms**
-   Median response time: **2,844 ms**
-   95th percentile: **7,509 ms**
-   Failed requests: **0**
-   Non-2xx responses: **67**

The run was stopped manually with `Ctrl+C`, so it was a partial test.

### Key concepts

-   **Concurrency:** Number of requests being processed at the same
    time.
-   **Throughput:** Number of requests completed per second.
-   **Latency:** Time taken for a request to complete.
-   **95th percentile:** 95% of requests completed within this time.
-   **Failed requests:** Requests ApacheBench classified as failed.
-   **Non-2xx:** Responses with status codes outside the 200--299 range;
    inspect the exact status codes.

### Auto Scaling validation

Monitor:

1.  ALB `RequestCount`
2.  Target group `RequestCountPerTarget`
3.  Target group `TargetResponseTime`
4.  `HTTPCode_Target_5XX_Count`
5.  EC2 `CPUUtilization`
6.  ASG `GroupInServiceInstances` and scaling activities

**Interview takeaway:** A successful load test is not just about
generating traffic. Correlate throughput and latency with target health,
EC2 metrics, and Auto Scaling activity to verify whether the
infrastructure responds correctly to increased demand.
