# Chapter 9 — Lab 2: APT & dpkg Package Troubleshooting & Security Investigation

## 📌 Overview
This lab provides hands-on practice with Debian package management tools (`dpkg` and `apt`) on **Kali Linux**. As a system administrator or SOC analyst, inspecting package states, repository health, candidate versions, and testing upgrade/removal impacts without altering production environments is a critical skill.

---

## 🛠️ Lab Tasks & Executed Commands

### Task 1 — Check the Package Database
Checks for incomplete, unmerged, or broken package installations in the `dpkg` database.
```bash
sudo dpkg --audit
```
* **Finding:** Returned clean execution (no output), confirming no half-configured or broken packages exist in the system database.

---

### Task 2 — Investigate a Package (`curl`)
Examines package status, binary location, and repository release candidate policy.
```bash
dpkg -s curl
command -v curl
apt-cache policy curl
```
* **Status:** `install ok installed`
* **Installed Version:** `8.20.0-5`
* **Candidate Version:** `8.22.0-1`
* **Binary Path:** `/usr/bin/curl`

---

### Task 3 — Check Repository Health
Refreshes APT's local package index without modifying installed applications.
```bash
sudo apt update
```
* **Result:** Successfully synchronized metadata across `kali-rolling` (main, contrib, non-free). Identifies **1,530 upgradable packages**.

---

### Task 4 — Preview Available Upgrades
Simulates package upgrades to inspect proposed changes safely using dry-run flags.
```bash
apt list --upgradable
apt upgrade --simulate
apt full-upgrade --simulate
```
* **Standard Upgrade:** Upgrades existing packages without removing dependencies.
* **Full Upgrade:** Evaluates dependency shifts, potentially adding or removing packages to resolve conflicts.

---

### Task 5 — Compare Removal vs. Purge
Simulates package uninstallation to highlight differences in configuration handling.
```bash
apt remove --simulate curl
apt purge --simulate curl
```
* **Remove:** Uninstalls the `curl` package and cascades dependent meta-packages (e.g., `kali-linux-default`), but leaves system configuration files in place.
* **Purge (`Purg` flag):** Deletes both the package binaries and all associated package-managed configuration files.

---

### Task 6 — Final Challenge Assessment Questions & Answers

| # | Question | Correct Answer | Explanation |
|---|---|---|---|
| **1** | What does `sudo apt update` do? | **Refreshes repository package lists** | Downloads latest package indexes; does not perform actual upgrades on disk. |
| **2** | Which command checks the installed status of a package? | **`dpkg -s curl`** | Queries `dpkg` database for installation state, version, and architecture. |
| **3** | Why use `apt full-upgrade --simulate`? | **To inspect proposed changes without performing the upgrade** | Allows safe dry-run evaluation of major dependency changes before execution. |
| **4** | What is the key difference between remove and purge? | **Purge also removes package-managed configuration files where applicable** | `remove` preserves configuration files; `purge` completely erases them. |

---

## 📂 Deliverables Included
* **PDF Report:** `linux_security_investigation_report.html` (Formatted multi-page document with embedded terminal screenshot evidence)
* **Zip Bundle:** `Chapter9_Lab2_Screenshots.zip` (Contains all high-resolution terminal capture images)