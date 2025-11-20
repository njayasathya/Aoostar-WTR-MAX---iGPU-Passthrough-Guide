# HDD SMART Testing Guide

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Quick Start](#quick-start)
3. [Understanding SMART](#understanding-smart)
4. [Running Tests](#running-tests)
5. [Verifying Results](#verifying-results)
6. [Troubleshooting](#troubleshooting)
7. [Best Practices](#best-practices)

---

## Prerequisites

### Check if smartctl is installed
```bash
which smartctl
```

### If not installed, install it:
**Ubuntu/Debian:**
```bash
sudo apt-get update
sudo apt-get install smartmontools
```

**RHEL/CentOS/Fedora:**
```bash
sudo yum install smartmontools
```

**macOS:**
```bash
brew install smartmontools
```

### Verify installation:
```bash
smartctl --version
```

---

## Quick Start

### 1. List all SMART-capable drives:
```bash
sudo smartctl --scan
```

### 2. Run short test on a drive:
```bash
sudo smartctl -t short /dev/sda
```

### 3. Check test status:
```bash
sudo smartctl -a /dev/sda | grep -A 5 "Self-test execution status"
```

### 4. View results:
```bash
sudo smartctl -H /dev/sda
```

---

## Understanding SMART

### What is SMART?
- **Self-Monitoring, Analysis and Reporting Technology**
- Built-in diagnostic system in modern HDDs
- Predicts drive failures before they occur
- Monitors drive health and performance

### SMART Health Status
- ✅ **PASSED** - Drive is healthy
- ❌ **FAILED** - Drive has critical issues
- ⚠️ **FAILING NOW** - Imminent failure

---

## Running Tests

### Test Types

#### 1. Short Self-Test (Recommended for new drives)
- **Duration:** 1-2 minutes
- **Scope:** Tests drive's firmware and electronics, reads sector list
- **Good for:** Quick verification of new drives

```bash
sudo smartctl -t short /dev/sda
```

#### 2. Extended/Long Self-Test (Comprehensive)
- **Duration:** 8-48+ hours (depending on drive size)
- **Scope:** Complete surface scan, tests every sector
- **Good for:** Thorough validation, detecting early failures

```bash
sudo smartctl -t long /dev/sda
```

#### 3. Conveyance Self-Test
- **Duration:** 5-20 minutes
- **Scope:** Tests damage from physical shock (transport)
- **Good for:** Drives that were just shipped/installed

```bash
sudo smartctl -t conveyance /dev/sda
```

### Running Tests on Multiple Drives

**One by one:**
```bash
sudo smartctl -t short /dev/sda
sudo smartctl -t short /dev/sdb
sudo smartctl -t short /dev/sdc
```

**All at once (parallel):**
```bash
for drive in sda sdb sdc sde sdf; do 
  sudo smartctl -t short /dev/$drive &
done
wait
```

### Aborting a Test

```bash
sudo smartctl -X /dev/sda
```

---

## Verifying Results

### 1. Quick Health Check
```bash
sudo smartctl -H /dev/sda
```

**Expected output:**
```
SMART overall-health self-assessment test result: PASSED
```

### 2. View Test Status (While Running)
```bash
sudo smartctl -a /dev/sda | grep -A 5 "Self-test execution status"
```

**Example output (in progress):**
```
Self-test execution status:      ( 249) Self-test routine in progress...
                                        90% of test remaining.
```

**Example output (completed):**
```
Self-test execution status:      (   0) The previous self-test routine completed
                                        without error or no self-test has ever 
                                        been run.
```

### 3. View Test Log/History
```bash
sudo smartctl -a /dev/sda | grep -A 20 "SMART Self-test log"
```

**Example output:**
```
SMART Self-test log structure revision number 1
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Short offline       Completed without error       00%        0         -
# 2  Short offline       Completed without error       00%        2         -
```

**Column explanations:**
- **Num:** Test number (most recent is #1)
- **Test_Description:** Type of test (Short, Extended, Conveyance)
- **Status:** Pass/Fail result
- **Remaining:** Completion percentage (00% = complete)
- **LifeTime(hours):** Hours on drive when test was run
- **LBA_of_first_error:** Location of first error (- = no errors)

### 4. Full SMART Information
```bash
sudo smartctl -a /dev/sda
```

This shows:
- Device model and serial number
- Firmware version
- Capacity and rotation speed
- Temperature
- Power-on hours
- Error count
- SMART attributes (detailed metrics)
- Self-test log

### 5. Full Report with Extra Details
```bash
sudo smartctl -x /dev/sda
```

More verbose output with NVMe/SCSI details if available.

---

## Interpreting Key SMART Attributes

### Critical Attributes to Monitor

| Attribute | Name | Threshold | What It Means |
|-----------|------|-----------|---------------|
| 05 | Reallocated Sectors Count | > 0 | Drive is failing, sectors being remapped |
| 187 | Reported UNCorrectable Errors | > 0 | Data corruption detected |
| 188 | Command Timeout | > 0 | Drive not responding properly |
| 194 | Temperature Celsius | > 55°C | Overheating (varies by drive) |
| 197 | Current Pending Sector Count | > 0 | Unreadable sectors found |
| 198 | Uncorrected Error Count (Offline) | > 0 | Uncorrectable errors exist |

**Green flags:**
- All attributes near 100 (Normalized Value)
- Low Power_On_Hours
- Low SMART Error Rate

---

## Checking Multiple Drives

### Status of all drives at once:
```bash
for drive in sda sdb sdc sde sdf; do 
  echo "=== /dev/$drive ==="; 
  sudo smartctl -H /dev/$drive; 
  echo ""; 
done
```

### Detailed info on all drives:
```bash
for drive in sda sdb sdc sde sdf; do 
  echo "=== /dev/$drive ==="; 
  sudo smartctl -a /dev/$drive | grep -E "Model|Serial|Capacity|Temperature|Power_On|Self-test execution status" -A 2; 
  echo ""; 
done
```

---

## Troubleshooting

### Issue: "Device does not support SMART"
```bash
sudo smartctl -s on /dev/sda
```

### Issue: "Command: Read from Input/Output error"
- Drive may be failing
- Check drive connection
- Try on different cable/port

### Issue: "SMART Health: FAILED"
- ❌ Do not use this drive for critical data
- Back up any data immediately
- Replace the drive

### Issue: Test takes longer than expected
- Normal for large drives (16TB+ can take 24+ hours)
- Don't interrupt unless absolutely necessary

### Issue: Cannot access drive without sudo
Add your user to the `disk` group:
```bash
sudo usermod -a -G disk $USER
newgrp disk
```

---

## Best Practices

### 1. New Drive Testing Protocol
```bash
# Step 1: Quick check
sudo smartctl -H /dev/sda

# Step 2: Run short test
sudo smartctl -t short /dev/sda
sleep 120
sudo smartctl -a /dev/sda | grep -A 10 "Self-test log"

# Step 3: Run long test overnight
sudo smartctl -t long /dev/sda

# Step 4: Next day - verify results
sudo smartctl -a /dev/sda | grep -A 10 "Self-test log"
```

### 2. Monitoring Schedule
- **New drives:** Immediate short + long tests
- **Production drives:** Weekly short tests
- **Critical systems:** Daily short tests
- **Aging drives:** Monthly long tests (if > 3 years)

### 3. Temperature Monitoring
```bash
sudo smartctl -a /dev/sda | grep -i "temperature"
```

Typical safe ranges:
- Normal: 25-45°C
- Warning: 45-55°C
- Critical: > 55°C

### 4. Automated Monitoring

**Set up cron job for daily checks:**
```bash
0 2 * * * /usr/sbin/smartctl -a /dev/sda >> /var/log/smartctl-sda.log 2>&1
```

### 5. Log Test Results
```bash
# Run test and save results
sudo smartctl -a /dev/sda > smart-results-sda-$(date +%Y%m%d).txt

# View saved results
cat smart-results-sda-20251120.txt
```

---

## Quick Reference Commands

| Task | Command |
|------|---------|
| List all drives | `sudo smartctl --scan` |
| Quick health check | `sudo smartctl -H /dev/sda` |
| Run short test | `sudo smartctl -t short /dev/sda` |
| Run long test | `sudo smartctl -t long /dev/sda` |
| Check test progress | `sudo smartctl -a /dev/sda \| grep "Self-test"` |
| View test history | `sudo smartctl -a /dev/sda \| grep -A 20 "Self-test log"` |
| Full SMART report | `sudo smartctl -a /dev/sda` |
| Very detailed report | `sudo smartctl -x /dev/sda` |
| Abort running test | `sudo smartctl -X /dev/sda` |
| Enable SMART | `sudo smartctl -s on /dev/sda` |
| Get temperature | `sudo smartctl -a /dev/sda \| grep -i temperature` |

---

## Real-World Example: Testing 5 New HDDs

```bash
#!/bin/bash
# Test script for 5 new drives

DRIVES=(sda sdb sdc sde sdf)

echo "Starting short tests on all drives..."
for drive in "${DRIVES[@]}"; do
  echo "Testing /dev/$drive..."
  sudo smartctl -t short /dev/$drive
done

echo "Waiting 2 minutes for tests to complete..."
sleep 120

echo "======== TEST RESULTS ========"
for drive in "${DRIVES[@]}"; do
  echo "=== /dev/$drive ==="
  sudo smartctl -a /dev/$drive | grep -A 15 "Self-test log"
  echo ""
done

echo "======== HEALTH STATUS ========"
for drive in "${DRIVES[@]}"; do
  echo "=== /dev/$drive ==="
  sudo smartctl -H /dev/$drive
  echo ""
done
```

Save as `test-all-drives.sh` and run:
```bash
chmod +x test-all-drives.sh
./test-all-drives.sh
```

---

## When to Replace a Drive

Replace immediately if:
- ❌ SMART Health shows FAILED
- ❌ Reallocated Sector Count > 0
- ❌ Uncorrectable Error Count > 0
- ❌ Temperature consistently > 60°C
- ❌ Multiple failed SMART attributes
- ❌ Test shows "Cannot open device" (physical failure)

---

## Additional Resources

- [smartmontools documentation](https://www.smartmontools.org/)
- [SMART attribute reference](https://en.wikipedia.org/wiki/Self-Monitoring,_Analysis_and_Reporting_Technology)
- [HDD SMART values guide](https://support.wdc.com/KnowledgeBase/answer.aspx?ID=11932)

---

**Last Updated:** November 20, 2025
**Version:** 1.0
