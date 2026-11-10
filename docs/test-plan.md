# Failure Test Plan and Expected Behaviour


## System under test

- 2 EC2 instances in private subnets across 2 Availability Zones, behind an Application Load Balancer
- Auto Scaling Group (min 2, max 4) with ELB health checks on `/health` (interval 15 s, 2 failures to mark unhealthy)
- CloudWatch alarms: high CPU, unhealthy hosts, low healthy hosts, 5xx errors, with SNS email alerts
- SSM Automation runbook, started by EventBridge when the unhealthy-hosts alarm fires:
  restart the app, wait up to about 90 s, then ask the Auto Scaling Group to replace the instance if still unhealthy
- Target-tracking scale-out policy on average CPU

## Test scenarios 

| # | Failure injected | How it is injected | Expected behaviour |
| --- | --- | --- | --- |
| 1 | App process killed | Open `/crash` once | Target unhealthy, alarm, restart or replacement | 
| 2 | App service stopped | `systemctl stop demoapp` via SSM shell | Same as 1 |
| 3 | Health endpoint failing, app running | `touch /opt/app/unhealthy` | Unhealthy, runbook clears the flag and restarts |
| 4 | Instance stopped | EC2 console, Stop | Auto Scaling Group replaces it | 
| 5 | Instance terminated | EC2 console, Terminate | Auto Scaling Group replaces it | 
| 6 | Instance in the other AZ terminated | EC2 console, Terminate | Users still served by the surviving instance |
| 7 | High CPU on both instances | Busy loops via SSM shell | CPU alarm, scale-out adds instances |
| 8 | Burst of 5xx errors | Refresh `/error` repeatedly | 5xx alarm and email only; no healing expected |
| 9 | Both apps stopped | `systemctl stop demoapp` on both | Total outage, then recovery |
| 10 | Restart cannot fix the fault | Break `app.py`, restart service | Runbook escalates to replacement |

## Known limits of this design (from the configuration, not from testing)

- No disk or memory metrics are collected, so disk-full and memory-pressure failures would not be detected.
- One NAT Gateway in one Availability Zone is a single point of failure for outbound access (a cost trade-off).
- HTTP only, no HTTPS or WAF.
- The Auto Scaling Group may replace an unhealthy instance before the runbook finishes its restart attempt; both outcomes count as recovery, but they are different healers.

## How measurements will be taken

- `probe.py` requests the load balancer once per second and logs status and serving instance; Enter marks the injection time.
- Detection time: failure injection to alarm `In alarm` (CloudWatch alarm history) or target `unhealthy`.
- Recovery time: failure injection to 2 healthy targets.
- Healer: Automation execution output, or Auto Scaling Group activity history.

