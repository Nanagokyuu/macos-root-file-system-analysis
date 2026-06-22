# macOS Golden Gate (Apple Silicon) Root File System, APFS Volume Group, and Boot Chain Deep Dive

**Languages:** [简体中文](README.md) [English](README_en.md) [日本語](README_jp.md)  
**Document Version:** v1.0  
**Subject:** MacBook Pro 14 (macOS Golden Gate 27.0 beta 1)  
**Analysis Method:** Hands-on verification (diskutil, mount, firmlink, snapshot, APFS Volume Group, Simulator Runtime)

---

# 1. Overview

The storage architecture of modern macOS is far more complex than it appears on the surface. To truly understand what is happening behind `/`, you first need to understand how Apple has evolved its layered system design over the years.

Starting with macOS Catalina, Apple split the system into:

* System Volume
* Data Volume

The motivation behind this change was to completely separate the read-only operating system files from user-writable data, laying the groundwork for subsequent security hardening.

Starting with Big Sur, Apple pushed even further, introducing more aggressive security mechanisms:

* Signed System Volume (SSV) — cryptographic signature verification for the system volume
* APFS Snapshot Boot — booting from a snapshot rather than mounting the volume directly
* Firmlink Namespace — cross-volume directory mapping implemented at the kernel level

On the Apple Silicon platform, because the chip has a dedicated security processor and firmware trust chain built in, the boot process gained several new participants:

* iSCPreboot — stores and enforces the local boot policy (LocalPolicy)
* xART — holds the trust cache and secure boot metadata
* Hardware — stores device activation, factory data, and other hardware-related information
* Secure Boot Policy — Apple Silicon's multi-tier secure boot policy framework

As a result, the root directory `/` of modern macOS no longer corresponds to any single physical disk partition. Instead, it is a unified namespace constructed jointly by multiple volumes. Understanding this is the prerequisite for everything else in this document.

---

# 2. APFS Container Structure

APFS (Apple File System) uses a "container" as its top-level storage unit, beneath which individual "volumes" reside. The primary APFS container on this machine lives at `disk3`, and contains the following volumes:

```text
disk3
├── disk3s1  System
├── disk3s2  Preboot
├── disk3s3  Recovery
├── disk3s5  Data
└── disk3s6  VM
```

The System volume and Data volume logically belong to the same "Volume Group":

```text
disk3s1
Name: Macintosh HD
Role: System

disk3s5
Name: Data
Role: Data
```

Both belong to:

```text
APFS Volume Group
UUID:
937449E6-EFC1-4FF8-B7B9-10B984769757
```

The Volume Group concept was introduced with macOS Big Sur. Its core purpose is to logically bind the System volume and Data volume together as a pair, so that Finder, diskutil, and system utilities can manage them as a single entity — without users ever needing to be aware of the underlying separation.

---

# 3. The Actual Boot Volume

A common misconception is that the system boots from `disk3s1` (the System Volume). That is not the case.

Verification:

```bash
mount | grep ' / '
```

Result:

```text
/dev/disk3s1s1 on /
(apfs, sealed, read-only)
```

Here we see a three-level device node `disk3s1s1`. It is not an independent partition — it is an **APFS Snapshot** on top of `disk3s1`. The full boot hierarchy is:

```text
disk3s1
    ↓
Snapshot
    ↓
disk3s1s1
    ↓
Mounted at /
```

Diagram:

```text
disk3s1
(System Volume)
    │
    ▼
Signed Snapshot
(disk3s1s1)
    │
    ▼
/
```

This means that `/System`, `/bin`, `/usr`, and other directories you see are actually coming from an immutable snapshot, not the live state of the System Volume itself.

---

# 4. Signed System Volume (SSV)

SSV is one of the core security features introduced in macOS Big Sur. Its goal is to ensure that system files cannot be silently tampered with by anyone — including root — after they leave the factory.

Verification:

```bash
diskutil info /
```

Result:

```text
Sealed: Yes
Volume Read-Only: Yes
```

The current boot is a:

```text
Signed System Volume
```

Its security properties operate on multiple levels:

* **Read-Only**: any write attempt is rejected
* **Apple Signature**: the entire volume's content is signed by Apple at release time
* **Hash Tree Verification**: the kernel validates the hash of every file it accesses, ensuring nothing has been modified
* **Secure Boot Verification**: iBoot validates the SSV signature before loading the system

Therefore, everything under the following paths:

```text
/System
/bin
/sbin
/usr
```

comes from:

```text
disk3s1s1
```

— not the Data volume. Even root cannot modify any file in these paths without breaking SSV integrity.

---

# 5. Root Namespace Union

When using macOS, users experience `/` as a single, unified file system. What is actually happening behind the scenes is a "Namespace Union" of two independent volumes.

Verification:

```bash
ls -ldO /
```

Result:

```text
sunlnk
```

And:

```bash
ls -ldO /System/Volumes/Data
```

Result:

```text
sunlnk
```

The `sunlnk` flag (Sun Link) is used by the macOS kernel to mark directory nodes participating in a Namespace Union, indicating that the system is using the:

```text
Namespace Union
```

mechanism. The logical structure of this mechanism is:

```text
System Snapshot
        +
Data Volume
        │
        ▼
Unified Root /
```

What the user ultimately sees is a merged view — read-only system files from the System Snapshot and writable user data from the Data Volume coexist in the same directory tree, with the boundary between them completely transparent.

---

# 6. Firmlink Mechanism

One of the key technologies enabling the Namespace Union is the **Firmlink**. A Firmlink is a cross-volume directory mapping mechanism Apple designed specifically for APFS. It works similarly to a symbolic link (Symlink), but differs fundamentally at the implementation level.

Verification:

```bash
cat /usr/share/firmlinks
```

Result:

```text
/Applications
/Users
/Library
/private
/opt
/cores
/pkg
/usr/local
...
```

Although these paths appear to live under the system root, they all actually point to:

```text
/System/Volumes/Data
```

For example:

```text
/Applications
        │
        ▼
/System/Volumes/Data/Applications
```

```text
/Users
        │
        ▼
/System/Volumes/Data/Users
```

```text
/Library
        │
        ▼
/System/Volumes/Data/Library
```

The core differences between a Firmlink and a traditional Symbolic Link are summarized below:

| Property         | Firmlink         | Symlink          |
| ---------------- | ---------------- | ---------------- |
| Kernel-level     | Yes              | No               |
| User-visible     | No               | Yes              |
| Cross-volume     | Yes              | No               |
| Finder display   | Normal directory | Link             |

Because Firmlinks operate at the kernel level and are invisible to users, the vast majority of applications accessing `/Applications` or `/Users` silently cross a volume boundary — completely unaware that they are reaching into Data Volume content.

---

# 7. Root Directory Structure

With the Firmlink mechanism in mind, we can now more clearly understand where the paths in the root directory actually live.

## 7.1 Directories Physically on the Data Volume

The following paths appear to be under the root directory, but are actually mapped to the Data Volume via Firmlinks:

```text
/Applications
/Users
/Library
/private
/opt
/cores
/pkg
```

Physically located at:

```text
/System/Volumes/Data
```

These are user-writable directories — the "mutable" portion of the system.

---

## 7.2 Symbolic Links

In addition to Firmlinks, the root directory also contains a number of traditional Symbolic Links, primarily for maintaining backwards path compatibility.

### etc

```text
/etc
    ↓
private/etc
```

---

### tmp

```text
/tmp
    ↓
private/tmp
```

---

### var

```text
/var
    ↓
private/var
```

`/private` is an actual directory residing in the System Snapshot, while `/etc`, `/tmp`, and `/var` are symbolic links pointing to its subdirectories. This design is also a historical legacy — in early versions of macOS, these paths behaved consistently with traditional Unix systems.

---

### home

Verification:

```bash
mount | grep home
```

Result:

```text
auto_home
```

Structure:

```text
/home
    ↓
/System/Volumes/Data/home
```

`/home` behaves a bit differently from the other paths. It is actually dynamically managed by:

```text
autofs
```

— responsible for auto-mounting network home directories on demand (such as user directories bound via Open Directory or LDAP). This is an on-demand mount mechanism.

---

# 8. The Volumes Directory

`/Volumes` is macOS's traditional mount-point directory; all visible disk volumes (including external storage) are typically mounted here. Notably, there is a design detail here that tends to confuse people.

Verification:

```bash
ls -ld /Volumes/"Macintosh HD"
```

Result:

```text
Macintosh HD -> /
```

So:

```text
/Volumes/Macintosh HD
```

is actually a:

```text
symbolic link
```

pointing back to:

```text
/
```

forming a circular structure:

```text
/Volumes/Macintosh HD
      │
      ▼
      /
```

This design is primarily for compatibility with older software (some applications reference paths in the form `/Volumes/Macintosh HD/...`) and for Finder's navigation logic — in the Finder sidebar, "Macintosh HD" is presented as an independent disk icon, and clicking it actually accesses the root directory `/`.

---

# 9. Data Volume

The Data Volume is the most frequently written volume in everyday use, carrying all the system's mutable runtime data.

Mounted at:

```text
disk3s5
```

Path:

```text
/System/Volumes/Data
```

Verification:

```bash
mount
```

Result:

```text
protect
root data
```

Indicating that this volume is the:

```text
Root Data Volume
```

Its primary responsibilities cover several areas:

* **User Data**: Home directories, documents, downloads, and all personal files
* **Third-Party Applications**: Apps installed from the App Store or other sources
* **Home Directory**: Every local user's home directory resides on this volume
* **Writable System Data**: Configurations and caches written at runtime by certain system components

Unlike the read-only System Volume, the Data Volume supports read-write operations and is the primary target of FileVault encryption.

---

# 10. Preboot Volume

The Preboot Volume is a key participant in the boot process, responsible for supplying the supporting materials needed to load the kernel.

Volume:

```text
disk3s2
```

Mounted at:

```text
/System/Volumes/Preboot
```

Contents stored here include:

```text
Kernel Collections   —— The kernel and kernel extension collection, built by kextcache
Boot Manifest        —— The boot manifest, describing the current boot configuration
Boot Support Files   —— Auxiliary files needed by iBoot and early boot stages
```

The Preboot Volume is normally not accessed directly by users after boot, but the system writes to it during OS updates or when rebuilding the kernel cache.

---

# 11. Recovery Volume

The Recovery Volume sits dormant most of the time, and is only activated and mounted when system recovery is needed.

Volume:

```text
disk3s3
```

Role:

```text
Recovery
```

Not mounted during normal operation; users cannot access it directly.

When the system fails to boot normally, or when the user explicitly enters Recovery Mode (hold the power button on Apple Silicon; hold Cmd+R on Intel), this volume is loaded and provides:

```text
macOS Recovery       —— A complete recovery environment
Restore System       —— Restore from Time Machine or the internet
Disk Utility         —— First Aid, partitioning, erasing, and more
Terminal             —— Command-line access for advanced troubleshooting
Reinstall macOS      —— Download and reinstall macOS from the internet
```

---

# 12. VM Volume

The VM Volume is dedicated to virtual memory management — it is where the system pages out memory under memory pressure.

Volume:

```text
disk3s6
```

Mounted at:

```text
/System/Volumes/VM
```

Primary functions:

```text
swapfile          —— Swap file for paging memory to disk
memory pressure   —— Works with the system's memory pressure management
Virtual Memory    —— Backs the unified virtual address space for all processes
```

This volume's mount parameters include:

```text
noexec
```

— meaning no executable files may be run from this volume, preventing potential security exploits (writing malicious code into swap and attempting to execute it).

---

# 13. Update Volume

The Update Volume serves as a temporary working space in the system update process, staging data before it takes effect in the live system.

Volume:

```text
disk3s4
```

Mounted at:

```text
/System/Volumes/Update
```

Its functions include:

```text
System Update Cache     —— Stores downloaded but not yet applied update packages
SSV Construction        —— Working space for building the new System Volume snapshot
Update Intermediate State —— Records update progress, enabling resumption after interruption
```

This design allows macOS to build a new system snapshot in the background without interrupting the running system, then switch to the new Signed Snapshot on the next reboot.

---

# 14. Apple Silicon Auxiliary Boot Container

The secure boot chain on Apple Silicon is significantly more complex than in the Intel era. Beyond the main `disk3` container, the system also has an additional APFS container:

```text
disk1
```

This container runs under the control of Apple Silicon's Secure Enclave and the boot auxiliary chip (SoC). Its structure is:

```text
disk1
├── iSCPreboot
├── xART
├── Hardware
└── Recovery
```

These four volumes together form the "secure boot auxiliary layer" unique to Apple Silicon, playing a critical role before iBoot formally loads the macOS kernel.

---

# 15. iSCPreboot

iSCPreboot (where iSC stands for "Internal SoC") is the core storage volume for Apple Silicon's boot policy.

Volume:

```text
disk1s1
```

Mounted at:

```text
/System/Volumes/iSCPreboot
```

When exploring its contents, one finds:

```text
937449E6-EFC1-...
```

This directory name matches the:

```text
APFS Volume Group UUID
```

exactly. This is no coincidence — it explicitly ties the contents of iSCPreboot to a specific Volume Group. Each registered macOS installation corresponds to one such directory.

Inside that directory is:

```text
LocalPolicy/
```

LocalPolicy is one of the central concepts in Apple Silicon's secure boot framework. It records the boot policy configured for this machine's current macOS installation, covering:

* **Startup Security**: Full Security, Reduced Security
* **External Boot Policy**: whether booting from external storage is permitted
* **Custom Kernel Extensions**: whether third-party kernel extensions are allowed to load
* **MDM Enrollment**-related policies

The overall directory structure of iSCPreboot is:

```text
iSCPreboot
├── VolumeGroupUUID
│   └── LocalPolicy
├── SystemRecovery
├── WiFi
└── SFR
```

---

# 16. xART

xART is one of the most opaque components in the Apple Silicon boot chain; neither its full name nor its internal implementation has been publicly documented by Apple.

Volume:

```text
disk1s2
```

Mounted at:

```text
/System/Volumes/xarts
```

Limited exploration reveals:

```text
A375F60A-....gl
```

Physical characteristics:

* Approximately 6 MB in size — extremely small
* Strictly protected by SIP (System Integrity Protection)
* Contents cannot be read even by root

Based on available reverse engineering findings and indirect hints from Apple's security white papers, its inferred purposes are:

```text
Trust Cache           —— Hash cache of system-trusted executables
Boot Artifacts        —— Intermediate artifacts required during the boot process
Secure Boot Metadata  —— Metadata supporting SSV verification
IMG4 / IM4P data      —— Data related to Apple's firmware signing format
```

Apple has yet to publish detailed documentation on xART; its internal format and access protocols remain a black box to outside researchers.

---

# 17. Hardware Volume

The Hardware Volume stores various hardware-related data accumulated over the device's lifetime — effectively the "archive room" for device identity and health status.

Volume:

```text
disk1s3
```

Mounted at:

```text
/System/Volumes/Hardware
```

Exploring its contents reveals the following directories and files:

```text
FactoryData          —— Baseline device data written at the factory
MobileActivation     —— Device activation information, related to Apple ID and iCloud
ProductDocuments     —— Product documents and certification materials
dramecc              —— DRAM ECC (error-correcting code) status records
recoverylogd         —— Recovery process logs
srvo                 —— Hardware service data with undisclosed purpose
```

This data represents the device's "factory-native properties" and is not modified during normal use, though the system may read or update it during device activation, hardware replacement, or certain repair scenarios.

---

# 18. Simulator Runtime

The iOS/iPadOS Simulator on macOS is not purely a software layer — its runtime is mounted as a standalone APFS volume. On this machine there are two distinct runtime mechanisms in play, reflecting an architectural transition Apple is actively pushing forward.

## 18.1 Traditional Runtime

```text
disk5s1
```

Mounted at:

```text
/Library/Developer/CoreSimulator/Volumes/iOS_23E254a
```

Corresponds to:

```text
iOS 26.4.1 Simulator
```

This is the traditional Simulator runtime mount approach: each iOS version's runtime exists as an independent APFS volume, mounted read-only to a corresponding path under `/Library/Developer/CoreSimulator/Volumes/`.

---

## 18.2 Cryptex Runtime

```text
disk7s1
```

Mounted at:

```text
/private/var/run/com.apple.security.cryptexd/mnt/...
```

Corresponds to:

```text
iOS 27.0 Simulator
```

Newer Simulator runtimes have begun adopting the:

```text
Cryptex
```

mechanism for dynamic mounting. Cryptex is a secure container format Apple introduced with macOS Monterey and iOS 15, originally developed for macOS security updates (Rapid Security Response) and since extended to Simulator runtime distribution and mounting, offering more flexible signature verification and on-demand loading. This transition signals that future Simulator Runtimes will all flow through the Cryptex channel, rather than the traditional static volume mount approach.

---

# 19. APFS Encryption Users

APFS supports multi-user encryption, and its user model does not map one-to-one to macOS login accounts. Understanding this distinction is essential for correctly working with FileVault.

Verification:

```bash
diskutil apfs listUsers /
```

Result:

```text
Local Open Directory User
Volume Owner: Yes

Personal Recovery User
Volume Owner: Yes
```

It is worth emphasizing: **APFS users are not the same as macOS login users.**

An APFS user is an entity that holds the authority to unlock the Volume Encryption Key (VEK). There are currently two Owner-level APFS users on this volume:

**Local Open Directory User**: Represents the encryption key holder for the local macOS account. When the user enters their login password, the key protector corresponding to this user is unlocked, which in turn decrypts the entire volume.

**Personal Recovery User**: Used in the following scenarios:

* When the user forgets their FileVault password, unlocking the volume via the Personal Recovery Key
* Authentication within the Recovery environment
* iCloud-managed recovery flows in conjunction with Apple ID

---

# 20. The Complete Boot Chain

Putting all the components above together, the complete boot chain for macOS on Apple Silicon looks like this. Each step is the trust foundation for the next; a verification failure at any point aborts the entire boot process:

```text
Boot ROM
    │  (hardwired into the SoC, Apple-signed, immutable)
    ▼
LLB
    │  (Low-Level Bootloader, first-stage bootloader)
    ▼
iBoot
    │  (second-stage bootloader, responsible for verifying subsequent components)
    ▼
iSCPreboot
    │
    └── LocalPolicy
            (reads and applies boot policy: security level, external boot permissions, etc.)
    │
    ▼
xART
    │
    └── Trust Cache
        Boot Artifacts
        (verifies the signature legitimacy of subsequently loaded components)
    │
    ▼
disk3s1
(System Volume)
    │
    ▼
Signed Snapshot
(disk3s1s1)
    │  (verifies SSV signature, validates Hash Tree integrity)
    ▼
Namespace Union
    │
    ├── System Snapshot    (read-only, from disk3s1s1)
    └── Data Volume        (read-write, from disk3s5)
    │
    ▼
Final Root /
```

---

# 21. Conclusion

From the analysis of each component above, one clear conclusion emerges:

The `/` of modern Apple Silicon macOS is not a single file system. It is a logical view jointly constructed by:

```text
Boot Policy
(iSCPreboot)
        +
Secure Boot Artifacts
(xART)
        +
Signed System Snapshot
(disk3s1s1)
        +
Root Data Volume
(disk3s5)
        +
Firmlink Namespace
        +
Symlink Layer
        +
Autofs
        │
        ▼
The / the end user sees
```

From a systems implementation perspective, the macOS root directory is in fact a logical file system view composed of **APFS Snapshot, Volume Group, Firmlink, Secure Boot, and Namespace Union** — not a conventional single disk partition in any traditional sense.

The core value of this design is that it achieves three things simultaneously without sacrificing user experience: cryptographic signature protection for system files, read-write flexibility for user data, and an unbroken chain of trust from firmware all the way up to the operating system. All three coexist behind what looks like an ordinary `/` — and that is precisely one of the most elegant aspects of the Apple Silicon security architecture.
