Since Jibo’s official servers were decommissioned, the robot hangs during the "Checking for Updates" phase, and of course. By sniffing its network traffic, (Community member : Jaked) has identified several hardcoded endpoints. Our goal is to redirect this traffic to a local server to simulate a "Success" response and unlock the robot’s full functionality.
- - -
Initial scans show that Jibo uses a standard network stack but maintains a strict internal firewall.

- **Primary DNS:** Hardcoded to `8.8.8.8` (Google DNS).
- **Web Config:** Ports `80`and `443` are closed/filtered by default.
- **ROS Bridge:** Port `9090` (used for MIT Workspace/SDK) is **blocked**. It appears to require a specific "Skill" (likely the _be-skill_) to be running before the port opens.

---

## Known Target Endpoints

The following table lists the IPs Jibo attempts to contact immediately after connecting to WiFi.

### **Time Synchronization (NTP)**

Jibo hits these IPs repeatedly to sync its internal clock. Without a successful time sync, SSL handshakes for other services will fail.

```
-45.115.225.48
216.229.4.66
93.57.144.50
66.231.64.28
194.195.253.58
```

### **Persistent AWS Infrastructure**

These are the most critical targets. Jibo hits these over and over, likely checking for signals or update manifests.

- **IP 1:** `35.172.208.31`
- **IP 2:** `44.198.39.206`

> **Hypothesis:** These were likely the primary API endpoints for `jibo.com`. I am investigating the packet payload to see if they are expecting JSON or XML responses.

### **Legacy & Third-Party Hits**

These appear less frequently, possibly for analytics or legacy newsletter/resource loading. 



| IP Address       | Ownership (potential) | Purpose                    |
| ---------------- | --------------------- | -------------------------- |
| `66.118.231.14`  | Constant Contact      | Maybe email or newsletter? |
| `172.232.15.202` | Akamai                | Content Delivery?          |
| `72.30.35.89`    | Yahoo                 | Weather / News API         |
| `67.210.96.32`   | Host Papa             | Domain hosting             |

---

## Current Observations

### **The Update Loop**

When Jibo reaches the "Checking for Updates" screen, it isn't "dead." Even while the screen is stuck loading , the background :

1. **Audio Processing stays active:** Saying _"Hey Jibo"_ causes the robot to turn toward the sound source.
2. **Network Persistence:** It continues to attempt connections to the AWS IPs listed above.
3. **Timeout Behavior:** It is unclear if the timeout is extremely long or if it is failing a specific SSL handshake.

### **Domain Discrepancy**

- **jibo.com:** Officially shut down; no longer resolves to an active site.
- **jibo.net:** Tribute site made by Community member Jibo-detective or RoboticaLabs on youtube

---

Check out [[Networking & ports & Error codes]] by ZaneDev from discord

---

