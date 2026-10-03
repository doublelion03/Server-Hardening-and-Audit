# Server-Hardening-and-Audit
carried out an ubuntu server hardening  and minimum audit of the entire system, the tools used here can of my choice and preference.




<h1>Ubuntu Linux Server Hardening & Security Validation</h1>

<h2>Project Overview</h2>

This project documents the security assessment, hardening, monitoring, and post-hardening validation of a self-deployed Ubuntu Linux server.

The objective is to establish a measurable security baseline, identify weaknesses in the server's configuration and attack surface, apply practical host-level security controls, and validate the resulting configuration using security auditing, access-control, integrity-monitoring, firewall, authentication, and auditing mechanisms.

The project followed a controlled:

<b>Baseline → Assess → Harden → Validate → Compare</b>

<h3>methodology.</h3>

The server was treated as a production-like asset, each control is evaluated according to its purpose, security effect, expected behavior, and evidence of implementation.


<h3>1. Security Objectives</h3>

The hardening exercise focuses on:

* Reducing unnecessary network exposure
* Restricting SSH password authentication
* Establishing host-based firewall controls
* Implementing mandatory access control through AppArmor
* Implementing security auditing through auditd
* Establishing file-integrity monitoring through AIDE
* Applying kernel-level security parameters
* Reviewing privileged filesystem permissions
* Reviewing running services and systemd isolation
* Establishing a measurable security baseline with Lynis
* Identifying remaining security controls requiring further work
* Producing before/after evidence suitable for technical review


<h3>2. Environment</h3>

**Operating System:** Ubuntu Server

**Deployment:** Self-deployed local server(Qemu/KNM)

**Primary security objective:** Host-level server hardening

**Assessment approach:** Local security assessment with controlled configuration changes and post-hardening validation.


<h3>3. Tools and Security Technologies</h3>

<b>3.1 Lynis</b>

**Purpose:**
Lynis is a Linux security auditing and hardening assessment tool. It examines operating-system configuration, authentication mechanisms, services, networking, filesystem permissions, logging, kernel parameters, and other security-related controls.

**Usefulness:**
Lynis provides a structured security assessment and produces warnings, recommendations, and a hardening index. I used it as the primary measurement mechanism for the before/after comparison.

**Project use:**

* Initial security baseline
* Identification of hardening opportunities
* Post-hardening assessment
* Measurement of security improvement


<b>3.2 UFW</b>

**Purpose:**
Uncomplicated Firewall (UFW) provides a simplified management interface for Linux firewall rules.

**Usefulness:**
It allows inbound and outbound traffic policies to be explicitly defined without requiring every firewall rule to be manually constructed with lower-level firewall tooling.

**Project use:**

The server was configured with:

* Default deny for incoming connections
* Default allow for outgoing connections
* Explicit SSH access permitted

This established a deny-by-default inbound security posture.

### Before
UFW was inactive.

<img width="731" height="572" alt="2026-10-02_08-48" src="https://github.com/user-attachments/assets/8a65549b-57e9-4fbc-ac76-20b59841c511" />


### After
UFW was enabled with a default-deny inbound policy.

<img width="777" height="683" alt="2026-10-02_10-20" src="https://github.com/user-attachments/assets/d32bf655-e896-41ed-afe5-35ba70355a78" />



<h3>4. SSH Hardening</h3>

## Purpose

SSH provides remote administrative access to the server. Because SSH is frequently exposed to network-based attacks, authentication configuration is a major security control.

## Initial Finding

The effective SSH configuration initially reported:

```text
passwordauthentication yes
pubkeyauthentication yes
permitrootlogin prohibit-password
```

<img width="766" height="588" alt="2026-10-02_09-42" src="https://github.com/user-attachments/assets/4c59ac07-ba99-4e6c-969e-ff4ec79dbe56" />


Although the main SSH configuration contained the desired password-authentication setting, an additional configuration file was overriding it:

```text
/etc/ssh/sshd_config.d/50-cloud-init.conf
```

The file contained:

```text
PasswordAuthentication yes
```

This explained why querying the effective configuration with:

```bash
sudo sshd -T
```

continued to report password authentication as enabled.

## Investigation

Cloud-init was checked because of the filename:

```text
50-cloud-init.conf
```

The server reported cloud-init as disabled.

This demonstrated an important Linux administration principle:

> The effective service configuration must be verified rather than assuming that a single configuration file represents the final configuration.

## Hardening Applied

Password authentication was changed to:

```text
PasswordAuthentication no
```

Public-key authentication remained enabled:

```text
PubkeyAuthentication yes
```

Root login remained:

```text
PermitRootLogin prohibit-password
```

The configuration was syntax-checked with:

```bash
sudo sshd -t
```

The effective configuration was then verified using:

```bash
sudo sshd -T | grep -E 'passwordauthentication|pubkeyauthentication|permitrootlogin'
```

### Expected hardened state

```text
passwordauthentication no
pubkeyauthentication yes
permitrootlogin prohibit-password
```

<img width="774" height="685" alt="2026-10-02_10-46" src="https://github.com/user-attachments/assets/f32085bc-830d-479b-b305-fce13faf8450" />


### Security effect

SSH password-based authentication is disabled, reducing exposure to password guessing and brute-force attacks.

SSH key authentication remains available for authorized administration.



<h3>5. AppArmor</h3>

## Purpose

AppArmor is a Linux Mandatory Access Control (MAC) framework.

Unlike traditional Unix permissions, which primarily control access based on users, groups, and file permissions, AppArmor can restrict what a particular application or process is permitted to access.

## Security value

If an application is compromised, AppArmor can limit the actions available to the compromised process.

This can reduce the impact of:

* Application compromise
* Unauthorized file access
* Privilege abuse
* Post-exploitation activity


Assess state using:

```bash
sudo aa-status
```

### Verification

The important evidence is whether profiles are loaded and whether relevant profiles are operating in enforce mode.
just note that once any porfile for an application or process in changed something has been altered and needs investigation if it wasnt intentional.

<img width="766" height="578" alt="2026-10-02_09-39" src="https://github.com/user-attachments/assets/a947febc-e00a-4110-8d86-c562df1c617a" />



<h3>6. auditd</h3>

## Purpose

`auditd` provides Linux security auditing.

It records security-relevant events such as:

* Authentication activity
* Changes to identity files
* Changes to privilege configuration
* SSH configuration changes
* PAM configuration changes
* Audit configuration changes

## Security value

Firewall logs tell us about network traffic.

Auditd provides a different layer:

> What security-relevant action happened on the operating system?

This makes it useful for investigation, accountability, and incident response.

## Configuration

Created Audit Rule for sensitive areas including:

```text
/etc/passwd
/etc/group
/etc/shadow
/etc/gshadow
/etc/sudoers
/etc/sudoers.d/
/etc/ssh/
/etc/security/
/etc/pam.d/
/etc/audit/
/var/log/auth.log
```

load rule:

```bash
sudo augenrules --load
```

verify:

```bash
sudo auditctl -l
```

<img width="695" height="192" alt="image" src="https://github.com/user-attachments/assets/e84ccb9f-3813-48c7-87eb-e7deb2dff105" />


## Behavioural verification

A controlled access to `/etc/passwd` was performed and the resulting event was searched using:

```bash
sudo ausearch -k identity -ts recent
```

<img width="713" height="678" alt="2026-10-02_16-34" src="https://github.com/user-attachments/assets/e3aa5433-6c3b-412b-9112-ce46a4d12b69" />


The resulting audit record contained fields including:

* CWD
* system call information
* operation result
* `success=yes`
* process/event information

This demonstrated that auditd was actively recording matching events as proven by the event time and specific newest event altering which it logged.



<h3>7. AIDE File Integrity Monitoring</h3>

## Purpose

AIDE (Advanced Intrusion Detection Environment) is a file-integrity monitoring system.

It creates a baseline describing selected filesystem objects using attributes such as:

* File permissions
* Ownership
* File size
* Timestamps
* Cryptographic hashes

A later scan compares the current filesystem against the trusted baseline.

## Security value

AIDE can help identify unauthorized or unexpected modification of monitored files.

It is particularly useful for detecting changes that may occur after compromise.

## Initialisation

AIDE was installed and its baseline database was initialized.

The resulting files included:

```text
/etc/aide/aide.conf
/etc/aide/aide.conf.d/
/etc/aide/aide.settings.d/
/var/lib/aide/aide.db
/var/lib/aide/aide.db.new
```

The baseline was subsequently used for an integrity check with:

```bash
sudo aide --config=/etc/aide/aide.conf --check
```

## Result

AIDE reported a difference involving:

```text
/var/log/sysstat/sa02
```

This does **not** constitute evidence of malicious modification.

`sa02` is a system activity accounting file that can legitimately change as the operating system continues collecting system statistics.

The changing size, timestamps, and cryptographic hashes therefore demonstrate that AIDE is capable of identifying a difference between the baseline and the current filesystem state.

## Important limitation discovered

Controlled test files created in `/tmp` and the user's home directory were not reported.

This indicates that those locations were not included in the active AIDE rules being used by the configuration.

Therefore, the correct conclusion is:

> AIDE was successfully deployed and performed baseline comparison, but its detection scope must be reviewed before claiming comprehensive filesystem integrity monitoring.

This is an important security-engineering finding rather than a failure of the tool.

[IMAGE PLACEHOLDER — AIDE baseline database present in `/var/lib/aide/`]

[IMAGE PLACEHOLDER — AIDE integrity-check output showing `/var/log/sysstat/sa02`]

[IMAGE PLACEHOLDER — AIDE configuration showing monitored filesystem rules]

---

# 8. Kernel Hardening

## Purpose

Linux kernel parameters control important networking and operating-system behaviors.

Security-sensitive parameters were reviewed and hardened where appropriate.

The hardening configuration included controls addressing:

* ICMP redirects
* Source routing
* Reverse-path filtering
* IPv6 redirects
* Kernel pointer exposure
* Kernel message visibility
* Process tracing
* Performance-event access

Configuration was placed in:

```text
/etc/sysctl.d/99-server-hardening.conf
```

The configuration was loaded using:

```bash
sudo sysctl --system
```

The active values were then verified directly through `sysctl`.

## Security value

These controls reduce certain classes of:

* Network manipulation
* Information disclosure
* Kernel debugging abuse
* Process inspection
* Network spoofing-related exposure

[IMAGE PLACEHOLDER — kernel hardening configuration file]

[IMAGE PLACEHOLDER — active `sysctl` values after hardening]

---

# 9. Kernel Lockdown Assessment

The kernel lockdown state was checked using:

```bash
cat /sys/kernel/security/lockdown
```

The server reported:

```text
none
```

This means kernel lockdown was not enabled.

This was **not automatically classified as a vulnerability**.

Kernel lockdown changes the operations available to privileged users and can interact with Secure Boot, virtualization, drivers, and legitimate administrative requirements.

Therefore, lockdown was treated as a separate control requiring environmental assessment rather than blindly enabling it.

---

# 10. Firewall Architecture

The final firewall model established the following baseline:

```text
                    Internet / Network
                           |
                           |
                    [ Ubuntu Server ]
                           |
                    UFW Firewall
                           |
             ---------------------------
             |                         |
        Inbound traffic           Outbound traffic
             |                         |
        DENY by default          ALLOW by default
             |
       Explicit exceptions
       for required services
```

This follows a least-exposure model.

A service is not automatically reachable simply because it is installed. Its required network port must be explicitly permitted.

---

# 11. Filesystem Permission Assessment

Privileged files were inventoried using SUID/SGID searches.

The objective was to identify executables that operate with elevated privileges.

The assessment did **not** automatically remove SUID/SGID permissions because many legitimate Ubuntu components depend on them.

This follows a safer principle:

> Identify first, determine necessity, then modify.

World-writable files were also inventoried for the same reason.

[IMAGE PLACEHOLDER — SUID/SGID inventory]

[IMAGE PLACEHOLDER — world-writable file assessment]

---

# 12. systemd Security Assessment

`systemd-analyze security` was used to assess service-level isolation.

The tool identified multiple services with `UNSAFE` or relatively high exposure scores.

This output was treated as an **assessment**, not as proof that each service was vulnerable.

A service may receive a poor systemd isolation score because it lacks sandboxing controls while still being required for normal system operation.

The appropriate remediation model is:

```text
Service
   ↓
Is it required?
   ↓
No → Disable/remove
   ↓
Yes
   ↓
Can it safely be sandboxed?
   ↓
Apply service-specific restrictions
```

Blindly hardening every service would create unnecessary availability risk.

[IMAGE PLACEHOLDER — `systemd-analyze security` baseline output]

---

# 13. Authentication Lockout

A brute-force protection control using PAM `pam_faillock` was investigated.

The server contained:

```text
pam_faillock.so
```

A lockout policy was attempted using:

```text
deny = 5
fail_interval = 900
unlock_time = 900
```

However, behavioral testing did not produce the expected five-attempt lockout behavior.

The control was therefore **not marked as successfully implemented**.

This remains a pending hardening item.

### Target behavior

```text
Repeated failed authentication
          ↓
Threshold reached
          ↓
Account temporarily locked
          ↓
Further authentication rejected
          ↓
Unlock after configured period
```

Further PAM-stack investigation is required before considering this control complete.

---

# 14. Security Assessment Results

## Lynis baseline

Initial Lynis assessment:

```text
Hardening Index: 61
Tests Performed: 270
```

The initial assessment identified, among other findings:

* Firewall not detected/active
* No intrusion-detection software detected
* No malware scanner detected
* Multiple host-level hardening opportunities

[IMAGE PLACEHOLDER — INITIAL LYNIS RESULT: Hardening Index 61]

---

## Post-hardening Lynis assessment

After applying the initial hardening controls:

```text
Hardening Index: 66
Tests Performed: 277
```

This represents a measurable increase in the Lynis hardening score.

The increase should not be interpreted as a universal measure of security. Lynis is an assessment framework, and its score reflects the controls and tests it evaluates.

The more important evidence is the individual configuration changes verified independently.

[IMAGE PLACEHOLDER — FINAL LYNIS RESULT: Hardening Index 66]

---

# 15. Before / After Security Comparison

| Control                       | Before                    | After / Current State           |
| ----------------------------- | ------------------------- | ------------------------------- |
| Host firewall                 | UFW inactive              | UFW active                      |
| Incoming traffic              | No UFW deny baseline      | Default deny                    |
| Outgoing traffic              | No UFW baseline           | Default allow                   |
| SSH password authentication   | Enabled                   | Disabled                        |
| SSH public-key authentication | Enabled                   | Enabled                         |
| Root SSH                      | `prohibit-password`       | `prohibit-password`             |
| AppArmor                      | Present                   | Assessed                        |
| auditd                        | Not established           | Active with custom rules        |
| Audit verification            | Not established           | Confirmed event recording       |
| AIDE                          | Not installed/initialized | Baseline created and checked    |
| Kernel parameters             | Baseline                  | Hardened parameters applied     |
| Kernel lockdown               | `none`                    | `none`, intentionally unchanged |
| SUID/SGID                     | Not assessed              | Inventoried                     |
| World-writable files          | Not assessed              | Inventoried                     |
| systemd isolation             | Not assessed              | Assessed                        |
| PAM lockout                   | Not verified              | Pending                         |
| Lynis score                   | 61                        | 66                              |

---

# 16. Evidence and Verification Strategy

The project deliberately used both **configuration evidence** and **behavioral evidence**.

Configuration evidence answers:

> Is the security control configured?

Behavioral evidence answers:

> Does the control actually behave as intended?

Examples:

### Firewall

Configuration:

```bash
sudo ufw status verbose
```

Behavior:

Unauthorized inbound traffic should be blocked unless explicitly permitted.

### SSH

Configuration:

```bash
sudo sshd -T
```

Behavior:

Password authentication should be rejected while authorized SSH-key authentication remains available.

### auditd

Configuration:

```bash
sudo auditctl -l
```

Behavior:

A controlled operation generated an auditable security event.

### AIDE

Configuration:

A baseline database exists.

Behavior:

AIDE identifies differences between the baseline and the current monitored filesystem.

### AppArmor

Configuration:

Profiles are loaded/enforced.

Behavior:

A confined process is prevented from performing actions outside its profile.

---

# 17. Security Lessons Demonstrated

## Effective configuration matters more than individual configuration files

The SSH investigation demonstrated that Linux services can load configuration from multiple locations.

The main configuration file initially suggested one policy while the effective SSH configuration reported another.

Using:

```bash
sshd -T
```

identified the actual behavior.

---

## Security tools must be validated behaviorally

Installing a tool is not equivalent to implementing security.

For example:

```text
auditd installed
        ≠
auditd confirmed working
```

The audit event test demonstrated that auditd was actually recording activity.

Likewise:

```text
AIDE database exists
        ≠
AIDE integrity monitoring proven
```

The AIDE comparison demonstrated that monitored filesystem differences could be detected.

---

## Security scores are indicators, not proof

The Lynis score increased from:

```text
61 → 66
```

However, the individual controls provide stronger technical evidence than the score alone.

---

# 18. Remaining Hardening Work

The following controls remain candidates for the next hardening phase:

* Correct PAM `pam_faillock` implementation
* Service-specific systemd sandboxing
* AIDE rule/scope refinement
* Review of unnecessary services
* Additional vulnerability scanning
* Network-based external validation with Nmap
* Centralized logging
* SOC integration
* Security-event correlation
* Intrusion detection
* Malware detection where justified
* Automated security-update strategy
* Backup and recovery validation

These were intentionally separated from the completed controls rather than being falsely marked as implemented.

---

# 19. Final Project State

The server progressed from an initial host-hardening baseline to a partially hardened configuration with independently verified controls.

The most significant verified changes were:

1. Host firewall activated with deny-by-default inbound policy.
2. SSH password authentication disabled.
3. SSH public-key authentication retained.
4. AppArmor confirmed as an existing security control.
5. auditd deployed with security-relevant audit rules.
6. auditd event generation and searching verified.
7. AIDE baseline database created.
8. AIDE filesystem comparison successfully executed.
9. Kernel security parameters hardened.
10. Privileged filesystem objects inventoried.
11. systemd service security assessed.
12. Lynis assessment performed before and after hardening.
13. Lynis hardening index increased from 61 to 66.
14. Remaining weaknesses were explicitly identified rather than being marked complete without verification.

---

# 20. Evidence Index

The following screenshots should be included in the GitHub project.

### Baseline Evidence

[IMAGE PLACEHOLDER — Ubuntu server system information]

[IMAGE PLACEHOLDER — Initial Lynis Hardening Index: 61]

[IMAGE PLACEHOLDER — Initial UFW status showing inactive]

[IMAGE PLACEHOLDER — Initial SSH effective configuration]

### Hardening Evidence

[IMAGE PLACEHOLDER — UFW configuration after hardening]

[IMAGE PLACEHOLDER — SSH configuration change]

[IMAGE PLACEHOLDER — Effective SSH configuration after hardening]

[IMAGE PLACEHOLDER — AppArmor status]

[IMAGE PLACEHOLDER — auditd rules]

[IMAGE PLACEHOLDER — auditd event showing successful recorded activity]

[IMAGE PLACEHOLDER — Kernel/sysctl hardening configuration]

[IMAGE PLACEHOLDER — Active kernel security parameters]

[IMAGE PLACEHOLDER — AIDE database initialization]

[IMAGE PLACEHOLDER — AIDE integrity check]

[IMAGE PLACEHOLDER — SUID/SGID assessment]

[IMAGE PLACEHOLDER — systemd security assessment]

### Final Evidence

[IMAGE PLACEHOLDER — Final Lynis Hardening Index: 66]

[IMAGE PLACEHOLDER — Final firewall status]

[IMAGE PLACEHOLDER — Final SSH effective configuration]

[IMAGE PLACEHOLDER — Final AppArmor status]

[IMAGE PLACEHOLDER — Final auditd status/rules]

---

# 21. Conclusion

This exercise demonstrates a practical Linux server-hardening workflow rather than a simple collection of security-tool installations.

The server was first assessed to establish a baseline. Security controls were then applied across network access, remote administration, mandatory access control, auditing, filesystem integrity, and kernel configuration.

The resulting configuration was independently re-tested.

The Lynis hardening index increased from **61 to 66**, while the number of tests increased from **270 to 277**. More importantly, individual controls were verified through their actual configuration and behavior.

The exercise also exposed several important limitations:

* AIDE's current monitoring scope does not cover every filesystem location.
* Kernel lockdown remains disabled and requires a separate risk assessment.
* PAM brute-force lockout has not yet been successfully verified.
* systemd's `UNSAFE` scores require service-by-service analysis rather than blanket configuration changes.
* Intrusion detection and centralized SOC monitoring remain future phases.

