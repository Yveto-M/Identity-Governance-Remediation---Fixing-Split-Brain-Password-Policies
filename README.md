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
<img width="531" height="292" alt="Screenshot 2025-12-30 135806" src="https://github.com/user-attachments/assets/e51aefe5-1474-492c-9bda-0cde78b8daa3" />

*> **Figure 3:** The "Smoking Gun." `MinPasswordAge` set to 1 day created a "denial of service" for the automation engine.*

### Step 2: Policy Remediation (GPO)
In a PAM environment, the automation engine must have the authority to change passwords instantly (e.g., in response to a detected compromise).

**Action:** I accessed the Default Domain Policy GPO and modified **Minimum Password Age** to **0 days**.

**GPO Modification**
<img width="508" height="272" alt="Screenshot 2025-12-30 140236" src="https://github.com/user-attachments/assets/6cc90f09-6e18-4422-a102-2764ebe4bc1d" />

*> **Figure 4:** Adjusting the Group Policy Object (GPO) to remove the time-based restriction.*

### Step 3: Enforcement
I forced an immediate policy update on the Domain Controller to apply the changes without a reboot.

**GPUpdate Command**
<img width="380" height="107" alt="Screenshot 2025-12-30 140334" src="https://github.com/user-attachments/assets/aa21a644-25c4-48ae-b9a2-86ea459d7d52" />


*> **Figure 5:** Executing `gpupdate /force` to ensure the new rules took effect immediately.*

---

## 🟢 Verification & Results
With the policy conflict resolved, I re-triggered the rotation.

### 1. Success Logs
The `pm.log` confirmed that the CPM successfully connected to the target, changed the password, and verified the new credential on **Try #0**.

**Successful Rotation Log**
<img width="953" height="272" alt="Screenshot 2025-12-30 141544" src="https://github.com/user-attachments/assets/6106b6ed-f337-4455-acf9-748f8bad03f5" />

*> **Figure 6:** The "Green" Log. `CACPM095I Change password on remote machine succeeded` confirms the barrier is removed.*

### 2. Final Artifact
The account now holds a high-entropy, machine-generated password that is fully managed by the vault.

**PVWA Verification**
<img width="436" height="234" alt="Screenshot 2025-12-30 142323" src="https://github.com/user-attachments/assets/6efb204a-567c-4844-bb9e-f3f805c98c87" />

*> **Figure 7:** Definitive Proof. The account `adm_recon` is now fully automated and secure.*

---

## 🧠 Engineering Takeaway
**"Automation requires Authority."** Simply installing a PAM tool is not enough. The underlying infrastructure (AD Group Policy) must be tuned to recognize the PAM system's authority. Leaving default settings like `MinPasswordAge = 1 Day` creates a vulnerability window where a compromised account cannot be rotated by the security team.
