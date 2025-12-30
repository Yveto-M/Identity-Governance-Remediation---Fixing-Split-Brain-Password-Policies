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

**Triggering Change**
<img width="609" height="334" alt="Screenshot 2025-12-30 135040" src="https://github.com/user-attachments/assets/f17cd2ed-2e30-44e2-9e7c-e82fa6d58d21" />

*> **Figure 1:** Initiating the manual rotation test via the PVWA portal.*

### 2. The Error
The `pm.log` (CyberArk Component Log) returned a specific error code: **`winRc=2245`**.
> *Error: The password does not meet the password policy requirements.*

**Policy Error Log**
<img width="973" height="229" alt="Screenshot 2025-12-30 135750" src="https://github.com/user-attachments/assets/40ba3dbc-4c22-4032-a98f-48b8ff7f0a6d" />

*> **Figure 2:** The Governance Conflict. The CPM generated a compliant password, but the Domain Controller rejected the change request.*

---

## 🔧 Technical Resolution

### Step 1: Root Cause Analysis (PowerShell)
I suspected the issue wasn't password *complexity*, but password *age*. I queried the Domain Controller's effective policy using PowerShell:

**Finding:** `MinPasswordAge` was set to `1.00:00:00` (1 Day). Because the account password had been set less than 24 hours ago, Active Directory was blocking the rotation to prevent "history churning."

**PowerShell Verification**
<img width="973" alt="Screenshot 2025-12-30 135750" src="https://github.com/user-attachments/assets/78393e83-9b81-4234-8c85-2e3d5f3032b4" />

*> **Figure 3:** The "Smoking Gun." `MinPasswordAge` set to 1 day created a "denial of service" for the automation engine.*

### Step 2: Policy Remediation (GPO)
In a PAM environment, the automation engine must have the authority to change passwords instantly (e.g., in response to a detected compromise).

**Action:** I accessed the Default Domain Policy GPO and modified **Minimum Password Age** to **0 days**.

**GPO Modification**
<img width="973" alt="Screenshot 2025-12-30 135750" src="https://github.com/user-attachments/assets/9d01b2a5-4f4a-4e2b-b5d1-6c2e7f8a9b3c" />

*> **Figure 4:** Adjusting the Group Policy Object (GPO) to remove the time-based restriction.*

### Step 3: Enforcement
I forced an immediate policy update on the Domain Controller to apply the changes without a reboot.

**GPUpdate Command**
<img width="973" alt="Screenshot 2025-12-30 135750" src="https://github.com/user-attachments/assets/1a2b3c4d-5e6f-7g8h-9i0j-k1l2m3n4o5p6" />

*> **Figure 5:** Executing `gpupdate /force` to ensure the new rules took effect immediately.*

---

## 🟢 Verification & Results
With the policy conflict resolved, I re-triggered the rotation.

### 1. Success Logs
The `pm.log` confirmed that the CPM successfully connected to the target, changed the password, and verified the new credential on **Try #0**.

**Successful Rotation Log**
<img width="973" alt="Screenshot 2025-12-30 135750" src="https://github.com/user-attachments/assets/a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6" />

*> **Figure 6:** The "Green" Log. `CACPM095I Change password on remote machine succeeded` confirms the barrier is removed.*

### 2. Final Artifact
The account now holds a high-entropy, machine-generated password that is fully managed by the vault.

**PVWA Verification**
<img width="973" alt="Screenshot 2025-12-30 135750" src="https://github.com/user-attachments/assets/q1w2e3r4-t5y6-u7i8-o9p0-a1s2d3f4g5h6" />

*> **Figure 7:** Definitive Proof. The account `adm_recon` is now fully automated and secure.*

---

## 🧠 Engineering Takeaway
**"Automation requires Authority."** Simply installing a PAM tool is not enough. The underlying infrastructure (AD Group Policy) must be tuned to recognize the PAM system's authority. Leaving default settings like `MinPasswordAge = 1 Day` creates a vulnerability window where a compromised account cannot be rotated by the security team.
