# Practice 4

**Student:** Iryna Velmakina · **Platform:** AWS Academy challenge lab · **Account:** AWS Academy sandbox · **Region:** us-east-1 · **Date:** 2026-11-23
**Load balancer:** `Cafe-ALB` · **Auto Scaling group:** `Cafe-ASG`

The café's single web server in one Availability Zone becomes a fleet of instances in
private subnets across two AZs, behind an Application Load Balancer, sized by CPU-based
target tracking. Built by hand in the console, so the evidence is screenshots — account
ID, ARNs and email redacted.

---

## 0. Architecture

**Before:** one `CafeWebAppServer` with a public IP in Public Subnet 1; one NAT gateway,
serving only Private Subnet 1.

**After:**

```
                  Internet
                     │ HTTP :80
          ┌──────────▼───────────┐   Cafe-ALB-SG: 80 from 0.0.0.0/0
          │       Cafe-ALB       │   Public Subnet 1 + Public Subnet 2
          └──────────┬───────────┘
                     │ Cafe-TG (HTTP :80)
        ┌────────────┴────────────┐
   us-east-1a                us-east-1b
   Private Subnet 1          Private Subnet 2
   webserver × n             webserver × n        ← Cafe-ASG (min 2 / max 6)
        │                         │
   Cafe-NAT-1                Cafe-NAT-2           ← outbound only, one per AZ
   (Public Subnet 1)         (Public Subnet 2)
```

---

## Task 1 — Inspecting the environment

### Network

| subnet | CIDR | AZ | route table | `0.0.0.0/0` → | auto public IP |
|---|---|---|---|---|---|
| Public Subnet 1 | 10.0.0.0/24 | us-east-1a | Public Route Table 1 | IGW | yes |
| Public Subnet 2 | 10.0.1.0/24 | us-east-1b | Public Route Table 2 | IGW | yes |
| Private Subnet 1 | 10.0.2.0/24 | us-east-1a | Private Route Table 1 | NAT (us-east-1a) | no |
| Private Subnet 2 | 10.0.3.0/24 | us-east-1b | Private Route Table 2 | **no route** | no |
| Private Subnet 3, 4 | 10.0.4–5.0/24 | us-east-1c, 1d | own tables | no route | no — not used |

Lab VPC `10.0.0.0/16`, internet gateway `Lab IGW`. Each subnet has its own route table,
so fixing Private Subnet 2 means editing its table — not creating a new one.

![Subnets](img/01-subnets.png)

### Security group, instance, AMI

![CafeSG inbound](img/02-cafesg-inbound.png)

![AMI](img/03-ami.png)

### Answers

| # | question | answer | why |
|---|---|---|---|
| 1 | Ports open in `CafeSG` | **80 only** | one inbound rule, TCP 80 from `0.0.0.0/0`. The description says *"Enable SSH, HTTP access"*, but there is no rule for 22 — rules decide, descriptions don't |
| 2 | Reach instances in Public Subnet 1 from the internet? | **yes** | route `0.0.0.0/0 → IGW` and public IPs on launch; the port still has to be open in the instance's SG |
| 3 | *Should* Private Subnet 1 reach the internet? | **yes** | outbound through NAT for updates and packages; nothing inbound |
| 4 | *Should* Private Subnet 2 reach the internet? | **yes** | it currently **can't** — no route. That gap is what Task 2 fixes. The question asks what the design needs, not what is configured |
| 5 | Connect to `CafeWebAppServer` from the internet? | **yes, over HTTP** | all three conditions hold: public IP, IGW route in Public Subnet 1, port 80 open in CafeSG. SSH is not open |
| 6 | AMI name | **Cafe WebServer Image** | EC2 → AMIs → Owned by me |

---

## Task 2 — NAT gateway for the second AZ

### Setup

| setting | value |
|---|---|
| name | `Cafe-NAT-2` |
| subnet | **Public Subnet 2** (us-east-1b) — a NAT needs an IGW route itself |
| availability mode | **Zonal** |
| connectivity | Public, new Elastic IP |

![Create NAT gateway](img/04-create-nat-2.png)

The console now defaults to a **Regional** NAT gateway — one gateway that spans all AZs.
Zonal was chosen on purpose: the lab's design is one NAT per AZ, and that is the pattern
being assessed.

### Route

**Private Route Table 2** → add `0.0.0.0/0 → Cafe-NAT-2`.

![Edit route](img/05-route-private-2.png)

![Route saved](img/06-route-saved.png)

The pre-existing NAT in Public Subnet 1 had no name; it was tagged `Cafe-NAT-1` for clarity.

![NAT gateways](img/07-nat-gateways.png)

### Why a second NAT instead of routing both subnets through the first

If Private Subnet 2 used the NAT in us-east-1a, losing us-east-1a would cut the internet
off from **both** AZs — the "highly available" fleet would share a single point of
failure. One NAT per AZ keeps the zones independent.

---

## Task 3 — Launch template

### Setup

| setting | value |
|---|---|
| name | `Cafe-LT` |
| AMI | My AMIs → `Cafe WebServer Image` |
| instance type | `t2.micro` |
| key pair | new: `cafe-key` |
| security group | `CafeSG` |
| resource tag | `Name = webserver` → Instances |
| IAM instance profile | `CafeRole` (Advanced details) |
| subnet | not set — the Auto Scaling group decides placement |

![Tags and IAM role](img/08-launch-template-tags-role.png)

![Launch template created](img/09-launch-template-created.png)

`CafeRole` is what makes Session Manager work in Task 6 — no key, no open port 22, no
bastion. The key pair was created because the lab requires it, but it was never used.

---

## Task 4 — Auto Scaling group

### Setup

| setting | value |
|---|---|
| name | `Cafe-ASG` |
| launch template | `Cafe-LT` (Default) |
| VPC | Lab VPC — the console pre-selects the *default* VPC, change it |
| subnets | **Private Subnet 1** (us-east-1a) + **Private Subnet 2** (us-east-1b) |
| load balancer | none at this step |
| desired / min / max | **2 / 2 / 6** |
| scaling policy | target tracking, **Average CPU utilization = 25 %** |
| instance warmup | 60 s |

![ASG created](img/10-asg-created.png)

### Result

Two `webserver` instances, one per AZ, **no public IPs**.

![Two webservers](img/11-two-webservers.png)

---

## Task 5 — Load balancer

### Setup

| resource | value |
|---|---|
| security group | `Cafe-ALB-SG`: inbound HTTP 80 from `0.0.0.0/0` |
| target group | `Cafe-TG`: instances, HTTP:80, Lab VPC, health check `/`, no targets registered by hand |
| load balancer | `Cafe-ALB`: internet-facing, IPv4, Public Subnet 1 + Public Subnet 2 |
| listener | HTTP:80 → forward to `Cafe-TG` |
| ASG integration | `Cafe-ASG` → Integrations → Load balancing → `Cafe-TG` |

![ALB created](img/12-alb-created.png)

![Target group attached to ASG](img/14-asg-tg-attached.png)

The ASG now registers every instance it launches in `Cafe-TG` and deregisters every
instance it terminates — nobody edits targets by hand.

### Result

![Targets healthy](img/13-tg-healthy.png)

| target | AZ | health |
|---|---|---|
| webserver #1 | us-east-1a | Healthy |
| webserver #2 | us-east-1b | Healthy |

---

## Task 6 — Testing load balancing and automatic scaling

### 6.1 Without load

`http://<Cafe-ALB DNS name>/cafe`

![Café via ALB](img/15-cafe-via-alb.png)

The request path is internet → ALB in a public subnet → instance in a private subnet.
The instances have no public IP, so this page can only come through the load balancer.

### 6.2 Under load

Session Manager into one web server (us-east-1a):

```console
$ sudo dnf install stress -y
...
Installed:
  stress-1.0.7-2.amzn2023.0.1.x86_64
Complete!

$ stress --cpu 1 --timeout 600
stress: info: [3352] dispatching hogs: 1 cpu, 0 io, 0 vm, 0 hdd
```

![Stress started](img/16-stress-started.png)

The `dnf install` succeeding is itself a check: the instance has no public IP, so the
package download went out through the NAT gateway.

### Result

![AlarmHigh in alarm](img/17-alarm-high.png)

![Activity history](img/19-activity-history.png)

![Desired capacity 5](img/18-asg-desired-5.png)

![Five webservers](img/20-five-webservers.png)

| time (UTC) | event | desired capacity |
|---|---|---|
| 00:16 | ASG created | 0 → 2 |
| 00:22 | stress started on one instance | 2 |
| 00:29:44 | `AlarmHigh` (CPU > 25 % for 3 datapoints in 3 min) → ALARM | **2 → 4** |
| 00:30:44 | alarm still in ALARM, next adjustment | **4 → 5** |

### Question — why 5 and not 6?

Target tracking aims to keep the **average** at 25 %. One instance at ~100 % among five
averages about 20 % — already below target, so the policy stops. The maximum of 6 is a
ceiling, not a goal.

### Question — why did the alarm take ~7 minutes?

The instances use **basic monitoring**: EC2 publishes CPU every 5 minutes, while the
alarm wants 3 one-minute datapoints above 25 %. The first breach becomes visible only
after the next 5-minute datapoint arrives. Detailed monitoring (1-minute metrics) would
cut the reaction time, at extra cost.

### Question — what happens after the stress test ends?

Scale-in is deliberately slower: `AlarmLow` needs CPU below the threshold for **15
datapoints in 15 minutes** before removing instances, and never below the minimum of 2.
Scaling out fast and in slowly avoids flapping when load is spiky — exactly the TV-show
case the café is preparing for.

---

## Files

```
lab4/
├── README.md
└── img/        console screenshots, redacted
```
