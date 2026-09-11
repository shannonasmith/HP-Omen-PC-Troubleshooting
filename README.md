<div align="center">

# 🔐 HP Omen Secure Boot & BitLocker Recovery

## Windows UEFI / TPM Troubleshooting & Root Cause Analysis

![Windows 11](https://img.shields.io/badge/Windows%2011-0078D4?style=for-the-badge&logo=windows&logoColor=white)
![UEFI](https://img.shields.io/badge/UEFI-Secure%20Boot-00599C?style=for-the-badge)
![BitLocker](https://img.shields.io/badge/BitLocker-TPM%202.0-107C10?style=for-the-badge&logo=microsoft)
![Troubleshooting](https://img.shields.io/badge/Hardware%20%26%20OS-Troubleshooting-6A1B9A?style=for-the-badge)

</div>

---

## 🧠 Scenario

A normally functioning **HP Omen 45L** running Windows 11 suddenly failed to boot following what appeared to be a routine background firmware/OS update. The first failure presented as a **Secure Boot violation**, reporting that a boot component had an invalid signature. On the following boot attempt, the system instead presented a **BitLocker Recovery** screen requesting the 48-digit recovery key.

At first, the symptoms appeared to point toward a potentially serious hardware or firmware problem. After recovering access, however, a second issue complicated the diagnosis: the system began intermittently entering a **POST loop when a Bluetooth headset USB dongle was connected during startup**.

The troubleshooting process therefore became two related but ultimately separate problems:

1. **Secure Boot / TPM / BitLocker failure following an update**
2. **USB peripheral causing a secondary POST enumeration problem**

Rather than immediately replacing hardware or assuming a failing CMOS battery, the system was diagnosed through **UEFI configuration, BitLocker recovery tools, PowerShell, and Windows Event Viewer**. The investigation demonstrated that the original BitLocker lockout was a security response to changed boot measurements — not evidence of a failed TPM, motherboard, or storage device.

---

## 🔥 Triggering Event

The BIOS update was located and downloaded through HP's own Software and Drivers portal.

<div align="center">
<img src="images/IMG_5852.JPEG" width="800">
</div>

*HP's Software and Drivers page for the OMEN 45L Gaming DT GT22-1167c PC.*

<div align="center">
<img src="images/IMG_5853.JPEG" width="800">
</div>

*HP Download and Install Assistant downloading an HP Consumer Desktop PC BIOS Update (SSID 8A98), alongside AMD and NVIDIA driver updates.*

A system information comparison before and after confirms the update actually changed the platform's firmware version:

<div align="center">
<img src="images/IMG_5691.JPEG" width="380"> <img src="images/IMG_5855.JPEG" width="380">
</div>

*System information before the update — BIOS F.12, 8/29/2023 · System information after the update — BIOS F.20, 4/20/2026*

This is the concrete triggering event referenced throughout the rest of this write-up: a documented, dated BIOS version change immediately preceding the failure.

---

## 🎯 Objective

* Restore boot access and recover the BitLocker-protected Windows installation
* Determine why Secure Boot and BitLocker rejected the boot environment
* Restore a secure UEFI/Secure Boot/BitLocker configuration
* Determine whether the secondary POST loop was a hardware failure
* Identify the actual root cause using system evidence, and document preventative steps

---

## 🖥️ Environment

| Component                    | Configuration                                                       |
| ---------------------------- | ------------------------------------------------------------------- |
| **System**                   | HP Omen 45L                                                         |
| **Operating System**         | Windows 11                                                          |
| **Firmware Interface**       | UEFI                                                                |
| **Secure Boot**              | Enabled                                                             |
| **TPM**                      | TPM 2.0                                                             |
| **Disk Encryption**          | BitLocker                                                           |
| **Boot Mode**                | UEFI                                                                |
| **Problem Peripheral**       | Bluetooth headset USB dongle                                        |
| **Primary Diagnostic Tools** | UEFI/BIOS, BitLocker Recovery Environment, PowerShell, Event Viewer |

### 📷 Physical System

<div align="center">
<img src="images/IMG_5689.JPEG" width="800">
</div>

*Internal view of the HP Omen 45L during the troubleshooting process.*

---

## 🚨 Initial Symptoms

### 1. Secure Boot Violation

The initial boot attempt produced a UEFI Secure Boot violation indicating that a boot component had an **invalid signature**.

<div align="center">
<img src="images/Picture1.jpg" width="800">
</div>

*Secure Boot Violation — "Invalid signature detected. Check Secure Boot Policy in Setup."*

There was no reason at this stage to assume the operating system itself was damaged — Secure Boot was doing exactly what it was designed to do: refusing to execute a boot component whose signature no longer matched the expected trust configuration.

---

### 2. BitLocker Recovery Prompt

A subsequent boot attempt reached the BitLocker recovery screen and requested the system's **48-digit recovery key**, which was not immediately available. The recovery screen also provided diagnostic information that became important later in the investigation.

<div align="center">
<img src="images/IMG_5686.JPEG" width="800">
</div>

*BitLocker Recovery — Additional recovery information showing `E_FVE_SECURE_BOOT_DISABLED` and a PCR 7 mismatch.*

The screen reported:

```text
Error category and code: Protector (E_FVE_SECURE_BOOT_DISABLED)

Mismatched PCR: 7
Expected digest: ccfc...a40e
Observed digest: 115a...02d8
Expected event label: SecureBoot
Observed event label: SecureBoot
Expected events count: 7
Observed events count: 6

Date/time of seal: 2026-07-06T03:57:41.977Z
```

The most significant detail was the **PCR 7 mismatch associated with SecureBoot** — PCR 7 records TPM measurements tied to Secure Boot policy and configuration, so this was direct evidence that the recovery event was related to a change in the measured Secure Boot state. The seal timestamp later became useful for correlating this event with the subsequent Windows and TPM events.

This created a second barrier to recovery: even if the Secure Boot issue could be bypassed, BitLocker was now protecting access to the Windows volume.

---

### 3. Secondary POST Loop

After the initial recovery, the system developed another intermittent problem: when the Bluetooth headset's USB dongle was connected during startup, the system would sometimes fail to progress through POST, repeatedly attempting to start rather than handing off to Windows. Because the system had already had one firmware-related boot problem, this initially raised the possibility of a motherboard, CMOS, or firmware fault — a hypothesis that ultimately proved incorrect.

---

## 🔎 Initial Hypothesis

The POST-loop behavior initially looked consistent with a **failing CMOS battery** or another firmware-level hardware problem — a reasonable first read for a system repeatedly failing at startup. On this particular machine, replacing the CMOS battery would also have meant removing the graphics card mount to access it, a genuinely tedious job worth avoiding if the evidence didn't actually support it.

The symptoms alone didn't provide enough evidence to justify replacing hardware, so the approach was:

> **Preserve the initial hypothesis, but test it against evidence before taking hardware action.**

This mattered even more once the BitLocker and Secure Boot events could be correlated with the timing of the update.

---

## 🩺 Ruling Out File Corruption

As an early diagnostic step, a system file integrity check was run to rule out corrupted Windows files:

```text
sfc /scannow
```

The scan completed with no integrity violations found — narrowing the field of possible causes before the Event Viewer investigation below identified the actual TPM/Secure Boot mechanism.

Notably, when this issue was described to a colleague with three years of professional IT support experience, she noted she'd never personally encountered this specific failure pattern — a fair sign this was a genuinely unusual scenario, not a routine, well-documented fix.

---

## 🛠️ Investigation & Recovery

### 🔓 Step 1 — Bypass Secure Boot

The first objective was simply to regain a boot path into the operating system. The system's UEFI configuration was accessed and **Secure Boot was temporarily disabled**, allowing the system past the signature validation failure. This was a deliberate temporary recovery step, not a permanent change — the goal was to investigate the underlying BitLocker and Windows state, not leave Secure Boot off.

---

### 🔐 Step 2 — Recover BitLocker Access

The next obstacle was the BitLocker recovery screen. The Windows Recovery Environment was accessed through **Advanced Options → Command Prompt**, and the following command temporarily suspended the BitLocker protector on the OS volume, allowing the system to proceed through recovery rather than requiring the unavailable recovery key:

```powershell
manage-bde -protectors -disable c:
```

The BitLocker recovery key was ultimately retrieved through the Microsoft account associated with the system, using a separate device. Once the key and system access were restored, investigation could continue from within Windows.

---

### 🔧 Step 3 — Restore the Security Baseline

Once Windows access was recovered, the objective shifted from **bypass** to **restoration** — Secure Boot should not remain disabled on a normal Windows 11 installation. The system was returned to UEFI configuration, and the Secure Boot keys were restored via **Secure Boot Configuration → Restore Factory Keys / Setup Defaults**. UEFI mode was confirmed, Legacy Support remained disabled, and Secure Boot was re-enabled:

```text
UEFI
  │
  ├── Secure Boot = Enabled
  │
  ├── TPM 2.0 = Enabled
  │
  └── Windows Boot Manager
          │
          └── BitLocker-protected Windows volume
```

---

### 💽 Step 4 — Verify BitLocker

After restoring the firmware configuration, BitLocker status was checked from Windows:

```powershell
Get-BitLockerVolume -MountPoint "C:"
```

<div align="center">
<img src="images/IMG_5690.JPEG" width="800">
</div>

*`Get-BitLockerVolume` output confirming the volume is FullyEncrypted with Tpm and RecoveryPassword key protectors, and Protection Status: On.*

The volume reported `VolumeStatus: FullyEncrypted`, and encryption was re-armed as necessary:

```powershell
Enable-BitLocker -MountPoint "C:" -EncryptionMethod XtsAes128 -UsedSpaceOnly
```

The objective here wasn't just getting Windows to boot — it was confirming the system returned to a properly encrypted and trusted state.

---

### 🔍 Step 5 — Investigate the Actual Cause

The system was operational again, but that doesn't answer the real troubleshooting question:

> **Why did this happen in the first place?**

Rather than assuming the TPM or motherboard had failed, Windows Event Viewer was used to review the BitLocker-related events surrounding the failure, at:

```text
Applications and Services Logs
└── Microsoft
    └── Windows
        └── BitLocker-API Management
```

**Event 898 — TPM Measurement Mismatch.** The TPM detected a mismatch between its stored security measurements and the measurements produced by the current boot configuration — the TPM was seeing a boot environment different from the one it had previously trusted. This aligned directly with the Secure Boot violation: the evidence pointed to a changed boot environment, not a failed TPM.

**Event 4103 — TPM Key Request Denied.** Windows made a silent TPM request for the BitLocker encryption key, and it was denied — because the TPM couldn't validate the current boot measurements against the trusted state, it refused to release the key automatically. This is exactly why BitLocker transitioned into recovery mode: not a separate unexplained failure, but the expected security response to an untrusted boot state.

**Event 793 — TPM Resealed Successfully.** BitLocker successfully resealed the volume to the TPM using the new boot measurements — confirming the TPM could establish a new trusted state once the boot configuration was restored.

---

## 🧩 Root Cause Analysis

The evidence supported a very different conclusion from the initial hardware-failure hypothesis:

1. A Windows/HP firmware update modified elements of the boot environment.
2. Secure Boot detected the resulting boot component signature no longer matched the trusted state.
3. The TPM detected the platform's boot measurements had changed — specifically, **PCR 7 no longer matched the value recorded when BitLocker was sealed.**
4. The TPM refused the automatic BitLocker key request, and BitLocker correctly entered recovery mode as a security fail-safe.
5. Once the configuration was restored, BitLocker successfully resealed against the new trusted measurements.

```text
Boot configuration change → Secure Boot trust mismatch → PCR 7 measurement mismatch → TPM refuses key release → BitLocker Recovery
```

There was also a complicating factor during the same recovery window: a **display signal/reconfiguration issue** appears to have obscured an OMEN firmware passcode-confirmation prompt tied to the Secure Boot change, which made the overall failure look more confusing than the underlying TPM/Secure Boot sequence actually was.

**The important distinction:** the evidence did *not* point to a failed TPM, SSD, motherboard, corrupted Windows installation, or dead CMOS battery — which mattered, because replacing hardware would not have fixed the actual problem.

---

## 🔌 Step 6 — Investigate the USB POST Loop

With BitLocker/Secure Boot resolved, the intermittent POST loop from the Bluetooth dongle remained. The first question was whether the system was simply trying to boot from the USB device — but the internal Windows Boot Manager was already prioritized ahead of USB devices, so simple boot-order misconfiguration didn't explain it.

### 🔬 USB Enumeration Hypothesis

The investigation shifted to **USB device enumeration during POST** — the OMEN firmware initializes connected USB hardware before handing off to Windows, and the headset's dongle appeared to cause the firmware to hang while enumerating it. This explained why the system booted fine without the dongle, why the problem occurred before Windows loaded, why reordering boot devices didn't help, and why the same peripheral worked fine once plugged in after Windows had already started.

The OMEN firmware didn't expose a traditional "USB Legacy Support" toggle — the relevant setting was **Advanced Boot → Options → USB Boot**, which was disabled.

### 📷 OMEN Firmware Evidence

<div align="center">
<img src="images/IMG_5687.JPEG" width="800">
</div>

*OMEN Setup Utility system log showing firmware-level startup information, reviewed during the POST-loop investigation.*

### ✅ Step 7 — Verify the USB Fix

The system was allowed to boot normally, and USB devices were reintroduced individually — the Bluetooth dongle connected fine once Windows had already loaded, confirming the problem was specific to **early USB enumeration during POST**, not a general USB or Windows fault. This secondary issue was treated as fully separate from the BitLocker/Secure Boot incident.

---

## 🔄 Recovery Verification

**Security:** Secure Boot re-enabled, UEFI mode confirmed, TPM functioning normally, BitLocker fully encrypted and resealed to current measurements, Windows booting normally.

**Hardware/POST:** System completes POST normally, Windows Boot Manager remains the boot target, USB Boot disabled in firmware, the Bluetooth dongle connects fine post-startup, and the POST loop no longer reproduces under normal use.

---

## 🔁 Recurrence — Second Occurrence

About two weeks after the initial recovery, the same failure pattern recurred following another firmware/OS update — a Secure Boot violation followed by a BitLocker recovery prompt. This was treated as a chance to test the root-cause model rather than a new mystery: the same steps (restore Secure Boot config, check BitLocker status, review the same 898 → 4103 → 793 event sequence) resolved it the same way.

The recurrence reinforced the original conclusion rather than undermining it — it confirmed that **any future firmware/BIOS update on this system can re-trigger the same TPM/Secure Boot measurement mismatch**, unless BitLocker is proactively suspended beforehand (see Lesson 1).

---

## 🛡️ Lessons Learned

**1. Suspend BitLocker before firmware changes.** `Suspend-BitLocker -MountPoint "C:" -RebootCount 1` before a BIOS/firmware update prevents a predictable change from unexpectedly triggering recovery.

**2. Keep more than one copy of the recovery key.** A Microsoft account is convenient, but a second copy (printed, or securely stored elsewhere) provides redundancy if the primary device or account isn't reachable.

**3. A boot loop's symptom doesn't identify its cause.** Firmware config, Secure Boot, boot-device enumeration, USB initialization, TPM state, bootloader issues, and actual hardware failure can all look identical from the outside. The CMOS hypothesis here was reasonable — the evidence just pointed elsewhere.

**4. Event Viewer and recovery screens are evidence, not noise.** The BitLocker-API Management events (898 → 4103 → 793) gave a far stronger diagnosis than just "the system won't boot."

**5. Secure Boot, TPM, and BitLocker each do something different.**

```text
Secure Boot → validates trusted boot components
TPM         → measures and validates platform state
BitLocker   → uses TPM trust to protect encryption keys
```

A Secure Boot change can indirectly trigger BitLocker recovery once the TPM's measurements no longer match the trusted boot configuration.

**6. OEM firmware doesn't always use standard terminology.** The OMEN firmware had no traditional "USB Legacy Support" option — the actual control lived under **Advanced Boot → Options → USB Boot**. Knowing a specific OEM's naming conventions matters when troubleshooting POST behavior.

---

## 💻 Command Reference

```powershell
# Rule out file corruption
sfc /scannow

# Temporarily disable BitLocker protectors (Recovery Environment)
manage-bde -protectors -disable c:

# Check BitLocker status
Get-BitLockerVolume -MountPoint "C:"

# Re-enable BitLocker
Enable-BitLocker -MountPoint "C:" -EncryptionMethod XtsAes128 -UsedSpaceOnly

# Suspend BitLocker before a firmware/BIOS update
Suspend-BitLocker -MountPoint "C:" -RebootCount 1
```

**Event Viewer path:** `Applications and Services Logs → Microsoft → Windows → BitLocker-API Management`

| Event    | Significance                                                             |
| -------- | ------------------------------------------------------------------------ |
| **898**  | TPM detected a mismatch between stored and current security measurements |
| **4103** | TPM denied a silent encryption-key request                               |
| **793**  | BitLocker successfully resealed the volume using the new measurements    |

---

## 📊 Key Takeaways

| Finding                 | Result                                                                        |
| ----------------------- | ------------------------------------------------------------------------------ |
| Secure Boot violation   | Boot signature/trust mismatch                                                  |
| BitLocker recovery      | TPM refused automatic key release                                              |
| PCR mismatch            | PCR 7 / SecureBoot measurement mismatch                                        |
| System file integrity   | No corruption found (`sfc /scannow`)                                           |
| TPM/CMOS failure?       | **No evidence of either — not supported by final evidence**                    |
| BitLocker state         | Fully encrypted, resealed successfully                                         |
| Secure Boot / UEFI      | Restored, enabled, confirmed                                                   |
| USB POST issue          | USB enumeration/boot behavior — USB Boot disabled                              |
| **System status**       | **Recovered; recurred once and resolved via the same diagnostic path; one residual symptom under investigation (see below)** |

---

## 🔎 Known Follow-Up Item — Frequent PIN Resets

Since the recovery, the affected Windows Hello PIN has needed to be reset more often than before the incident — three times as of this writing. A likely explanation: this system uses AMD's firmware-based **fTPM** rather than a discrete TPM chip, and a BIOS/firmware update can sometimes reset the fTPM's stored state as a side effect of flashing new firmware. If that's happening here, it would explain both symptoms as one event rather than two — BitLocker recovery because its key seal is invalidated, and the PIN reset because Windows Hello's credential is *also* TPM-bound, independent of BitLocker.

This hasn't been confirmed with the same rigor as the primary incident yet, and is being tracked as an open item.

---

## 💡 Skills Demonstrated

Windows 11 troubleshooting • UEFI firmware configuration • Secure Boot troubleshooting • TPM 2.0 analysis • BitLocker recovery & command-line administration • PowerShell • Windows Recovery Environment • Windows Event Viewer & event correlation • Root cause analysis • POST/USB enumeration troubleshooting • Evidence-based troubleshooting • Security configuration validation • Preventative maintenance planning

---

## 🧠 Final Assessment

This recovery is a good example of why complex boot failures deserve an **evidence-gathering approach rather than a component-swapping one**. The initial symptoms made hardware failure look plausible, especially once the POST loop appeared — but the BitLocker and TPM event sequence told a clearer story: the Secure Boot violation changed the platform's measured boot state, the TPM refused to release the BitLocker key because those measurements no longer matched, and BitLocker correctly entered recovery. Once the firmware configuration was restored, the TPM resealed successfully.

The USB POST loop turned out to be a separate, firmware-level enumeration issue, resolved by disabling USB Boot. The same root cause resurfaced once more two weeks later after another firmware update, resolving via the identical diagnostic path — and one residual symptom (frequent PIN resets) remains open rather than fully closed.

**The key lesson:** don't replace hardware just because the symptoms look like hardware failure. Establish the timeline, collect the evidence, correlate the events, and let the system tell you what actually happened.

---

<div align="center">

## 👤 Shannon Smith

Cybersecurity | SOC Operations • Detection Engineering • Incident Response

</div>
