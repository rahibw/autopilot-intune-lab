# Windows Autopilot Lab: Zero-Touch Enrollment with Intune and Entra ID

A hands-on lab where I registered a Windows 11 VM in Windows Autopilot, grouped it automatically, applied a deployment profile, and watched it join Microsoft Entra ID and enroll in Intune with a single sign-in. I built this while studying for the Microsoft MD-102 (Endpoint Administrator) exam.

## What this lab shows

- Registering a device in Autopilot by capturing its hardware hash
- Using a dynamic Entra ID group to place registered devices into a group automatically
- Building an Autopilot deployment profile that controls the out-of-box experience (OOBE)
- Building an Enrollment Status Page (ESP) so the user can see setup progress
- Testing the full flow from OOBE through to Intune enrollment

## Environment

| Item | Detail |
|---|---|
| Device | Windows 11 virtual machine (VMware) |
| Tenant | Microsoft 365 / Intune trial tenant |
| Licensing | Intune and Microsoft Entra ID P1 (dynamic groups require P1) |
| Tools | PowerShell, Get-WindowsAutopilotInfo, Intune admin center, Entra admin center |

## How Autopilot works (in short)

1. The device boots into OOBE and connects to the internet.
2. It contacts the Autopilot service and sends its hardware hash.
3. Microsoft matches the hash to the organization (tenant) it is registered to.
4. The device downloads the assigned Autopilot deployment profile.
5. The user signs in on the organization's branded sign-in screen.
6. The device joins Microsoft Entra ID and enrolls in Intune.
7. The Enrollment Status Page tracks the policies and apps Intune applies, and the user reaches the desktop when it finishes.

## Steps

### 1. Capture the hardware hash

Run on the device in an elevated PowerShell window (in OOBE, press Shift+F10 and type `powershell`):

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
Install-Script -Name Get-WindowsAutopilotInfo
Get-WindowsAutopilotInfo -OutputFile HardwareID.csv
```

### 2. Register the device in Autopilot

In the Intune admin center: **Devices → Enrollment → Windows Autopilot → Devices → Import**, select the CSV, then **Sync** until the device appears.

### 3. Create a dynamic device group

In the Entra admin center: **Groups → New group**. Group type **Security**, membership type **Dynamic Device**, and this rule:

```
(device.devicePhysicalIDs -any (_ -startsWith "[ZTDId]"))
```

This matches every device registered in Autopilot. To target a specific batch, use a Group Tag instead:

```
(device.devicePhysicalIds -any (_ -eq "[OrderID]:LabVM"))
```

### 4. Create the deployment profile

In Intune: **Devices → Enrollment → Windows Autopilot → Deployment profiles → Create profile → Windows PC**, then assign it to the dynamic group.

| Setting | Value | Why |
|---|---|---|
| Convert all targeted devices to Autopilot | No | Only needed for devices already enrolled in Intune but not registered; this device was registered by hash |
| Deployment mode | User-driven | The user signs in during OOBE |
| Join to Microsoft Entra ID as | Microsoft Entra joined | Cloud-only join |
| Microsoft Software License Terms | Hide | Skips the EULA page |
| Privacy settings | Hide | Privacy is set by Intune policy instead |
| Hide change account options | Hide | Keeps the user on the work sign-in |
| User account type | Standard | Least privilege; the user is not a local admin |
| Allow pre-provisioned deployment | No | Not needed for this lab |
| Device name template | `NYC-%RAND:5%` | Names devices automatically (15 characters max) |

### 5. Create an Enrollment Status Page

In Intune: **Devices → Enrollment → Windows Autopilot → Enrollment Status Page → Create**, then assign it to the same group. Key settings:

- Show app and profile configuration progress: Yes
- Show an error if installation takes longer than 60 minutes
- Turn on log collection and diagnostics for end users
- Only show the page to devices provisioned through OOBE
- Install Windows updates (might restart the device)
- Block device use until all apps and profiles are installed

### 6. Test in OOBE

The VM had already been through setup, so I sent it back to OOBE with Sysprep:

```
C:\Windows\System32\Sysprep\sysprep.exe /oobe /generalize /reboot
```

(Settings → System → Recovery → Reset this PC also works.)

## Results

- At the sign-in screen, the organization's branding appeared before any credentials were entered, and there was no option to create a local account.
- After signing in, the Enrollment Status Page showed setup progress. The EULA, privacy, and account setup pages were skipped.
- The device joined Entra ID, enrolled in Intune, prompted for a Windows Hello PIN, and was named **NYC-81247** by the template.
- In Intune, the device shows as Corporate and Compliant with the signed-in user as primary user, and the Autopilot device list shows its profile status as **Assigned**.

## Troubleshooting notes

**Sysprep validation error.** Sysprep failed on the first attempt with "Sysprep was not able to validate your Windows installation." It records the reason in `C:\Windows\System32\Sysprep\Panther\setupact.log`. A common cause on Windows 11 is a Store app installed for a user but not provisioned for all users. Reading that log is the way to find the exact cause.

## What I learned

- The hardware hash is what lets a device identify its organization before anyone signs in.
- Autopilot mainly controls the OOBE. The join and enrollment happen when the user signs in, and Intune applies the policies and apps afterward.
- The deployment profile and the Enrollment Status Page are separate profiles; they are connected only through group assignment.
- Manual enrollment (choosing "work or school" in OOBE) also works, but Autopilot makes the process consistent and keeps users from picking a different path.

## Next steps

- Deploy a required app so the Enrollment Status Page has something to track
- Use Group Tags to apply different profiles to different device types
- Try pre-provisioning on hardware with a TPM
- Compare manual enrollment to Autopilot side by side

## Full write-up

The complete step-by-step document with screenshots is in this repository: `Autopilot-Registration-Lab.pdf`
