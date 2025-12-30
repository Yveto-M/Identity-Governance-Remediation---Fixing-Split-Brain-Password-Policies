# 🛡️ Project: Identity Governance Remediation - Fixing "Split-Brain" Password Policies

### 🚀 Executive Summary
This project documents the diagnosis and resolution of a critical failure in the automated password rotation lifecycle of a privileged account. While the CyberArk PAM service was operational, it failed to rotate credentials due to a hidden conflict between the **CyberArk Platform Policy** and the **Active Directory Domain Policy**.

**Objective:** Align Identity Governance policies to ensure the Central Policy Manager (CPM) has full authority to rotate credentials instantly during a threat event.

**Skills Demonstrated:** Log Analysis, Active Directory GPO Management, PowerShell Troubleshooting, IAM Policy Tuning.

---

## 💥 The Incident (Policy Collision)
After onboarding a new Domain Admin account (`adm_recon`), I initiated a manual password rotation to verify the automation loop. The operation failed immediately.

### 1. The Trigger
I manually requested a password change via the PVWA portal to test the rotation engine.

![Triggering Change](images/Screenshot%202025-12-30%20135040.png)
*> **Figure 1:** Initiating the manual rotation test via the PVWA portal.*

### 2. The Error
The `pm.log` (CyberArk Component Log) returned a specific error code: **`winRc=2245`**.
> *Error: The password does not meet the password policy requirements.*

![Policy Error Log](images/Screenshot%202025-12-30%20135750.png)
*> **Figure 2:** The Governance Conflict. The CPM generated a compliant password, but the Domain Controller rejected the change request.*

---

## 🔧 Technical Resolution

### Step 1: Root Cause Analysis (PowerShell)
I suspected the issue wasn't password *complexity*, but password *age*. I queried the Domain Controller's effective policy using PowerShell:
```powershell
Get-ADDefaultDomainPasswordPolicy
