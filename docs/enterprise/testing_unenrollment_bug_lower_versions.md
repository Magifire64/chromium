# Testing for Unenrollment Bug on Lower ChromeOS Versions

## Overview

This guide helps system administrators test whether their ChromeOS devices on older versions are affected by the spontaneous unenrollment bug.

## Quick Start - 5 Minute Test

### What You Need
- Access to enrolled ChromeOS devices (managed fleet)
- Root/developer shell access OR ability to view chrome://policy
- 5-10 minutes

### Steps

#### Option 1: Using Chrome UI (Easiest - No Root Required)

1. **On an enrolled device**, open Chrome browser
2. **Navigate to**: `chrome://policy`
3. **Check if device shows**:
   - ✅ "Device policies" section populated with policies
   - ✅ "Refresh policies" button works
   - ✅ No error messages about enrollment

4. **Navigate to**: `chrome://histograms`
5. **Search for**: `Enterprise.EnrolledPolicyHasDMToken`
6. **Look at the values**:
   - If you see samples with value=0 (false), this indicates the bug occurred
   - If only value=1 (true), device is not affected

7. **Reboot the device** and repeat steps 2-6
8. **If you see enrollment screen** after reboot on a device that was enrolled: **BUG DETECTED**

#### Option 2: Using Developer Shell (Most Detailed)

1. **Enable developer mode** on a test device (WARNING: Wipes data)
   - Power off device
   - Hold Esc + Refresh, press Power
   - Press Ctrl+D at recovery screen
   - Follow prompts to enable developer mode

2. **After enrollment**, press `Ctrl+Alt+T` to open crosh
3. Type `shell` to get a shell prompt
4. **Run diagnostics**:
   ```bash
   # Check if enrollment recovery flag is set
   cat /home/chronos/Local\ State 2>/dev/null | grep -o '"EnrollmentRecoveryRequired":[^,}]*'
   
   # Check for DM token errors in logs
   sudo tail -1000 /var/log/chrome/chrome | grep -i "no DM token"
   
   # Check device enrollment status
   sudo cryptohome --action=install_attributes_get --name=enterprise.device_id
   ```

5. **Interpret results**:
   - If `EnrollmentRecoveryRequired:true` but device is enrolled: **BUG DETECTED**
   - If you see "no DM token" errors repeatedly: **BUG DETECTED**
   - If device_id is empty but device shows as enrolled: **BUG DETECTED**

## Detailed Testing Scenarios

### Scenario 1: Test During High Load

**Purpose**: Simulate conditions that trigger the bug (slow I/O, delayed loading)

**Steps**:
1. Enroll a test device
2. **Create high system load**:
   ```bash
   # In developer shell
   # Note: The 'stress' tool may need to be installed first
   # Alternative: Open multiple chrome tabs and applications to create load
   stress --cpu 4 --io 4 --vm 2 --vm-bytes 128M --timeout 60s &
   ```
3. **Immediately reboot** during the stress test
4. **Check after reboot** if enrollment recovery was triggered
5. **Expected (with bug)**: Device may show enrollment screen
6. **Expected (without bug)**: Device boots normally to login screen

### Scenario 2: Test with Slow Storage

**Purpose**: Test devices with slower eMMC/HDD storage

**Steps**:
1. Select test devices known to have slower storage
2. Enroll the devices normally
3. Reboot multiple times (5-10 reboots)
4. **Monitor** for enrollment recovery triggers
5. **Record** how many reboots trigger false recovery

### Scenario 3: Test Fleet-Wide Pattern

**Purpose**: Identify if certain device models are more affected

**Steps**:
1. **Identify enrolled devices** in your fleet
2. **Query admin console** or device management system for:
   - Devices that recently went through enrollment recovery
   - Devices with enrollment status changes
   - Devices with policy sync errors

3. **Correlate with**:
   - Device model/hardware
   - ChromeOS version
   - Recent OS updates
   - Reboot frequency

4. **Look for patterns**:
   - Do certain models show higher recovery rates?
   - Do devices with specific OS versions trigger more often?
   - Does recovery correlate with OS updates?

## Comprehensive Test Suite

### Test 1: Fresh Enrollment Test

```bash
# Test Script - Run on test device in developer mode

#!/bin/bash
echo "=== ChromeOS Unenrollment Bug Test ==="
echo "Test 1: Fresh Enrollment"

# 1. Check current state
echo "Checking current enrollment state..."
sudo cryptohome --action=install_attributes_get --name=enterprise.enrolled
echo ""

# 2. Enroll device (manual step - pause here)
echo "Please enroll this device through normal enrollment flow"
echo "Press ENTER after enrollment completes"
read

# 3. Verify enrollment
echo "Verifying enrollment..."
DM_TOKEN=$(sudo cryptohome --action=install_attributes_get --name=enterprise.device_id)
if [ -z "$DM_TOKEN" ]; then
  echo "❌ FAIL: Device not properly enrolled"
  exit 1
fi
echo "✅ Device enrolled with DM token"

# 4. Check initial state
echo "Checking Local State for recovery flag..."
RECOVERY=$(cat /home/chronos/Local\ State 2>/dev/null | grep -o '"EnrollmentRecoveryRequired":true')
if [ ! -z "$RECOVERY" ]; then
  echo "❌ FAIL: Recovery flag set on fresh enrollment"
  exit 1
fi
echo "✅ No recovery flag set"

# 5. Simulate multiple policy updates
echo "Simulating policy updates..."
for i in {1..10}; do
  dbus-send --system --type=method_call --print-reply \
    --dest=org.chromium.SessionManager /org/chromium/SessionManager \
    org.chromium.SessionManagerInterface.RefreshDevicePolicy
  sleep 2
done

# 6. Check again
RECOVERY=$(cat /home/chronos/Local\ State 2>/dev/null | grep -o '"EnrollmentRecoveryRequired":true')
if [ ! -z "$RECOVERY" ]; then
  echo "❌ FAIL: Recovery flag set after policy updates (BUG DETECTED)"
  exit 1
fi
echo "✅ No recovery flag after policy updates"

# 7. Reboot test
echo "Reboot test - will reboot in 10 seconds"
echo "After reboot, manually check if device shows enrollment screen"
echo "If it goes to login screen (not enrollment), test PASSED"
echo "If it shows enrollment screen, BUG DETECTED"
sleep 10
# Note: After reboot, you'll need to check manually
sudo reboot
```

### Test 2: Multi-Reboot Stress Test

**Note**: This test requires manual intervention after each reboot or setting up as a startup service.

```bash
#!/bin/bash
# Run this script on multiple test devices
# After each reboot, manually re-run the script or set it up as a startup service

LOG_FILE="/tmp/enrollment_bug_test.log"
STATE_FILE="/tmp/reboot_test_state"

# Initialize or read state
if [ -f "$STATE_FILE" ]; then
  CURRENT_REBOOT=$(cat $STATE_FILE)
else
  CURRENT_REBOOT=0
fi

REBOOT_COUNT=10
NEXT_REBOOT=$((CURRENT_REBOOT + 1))

echo "=== Multi-Reboot Stress Test ===" | tee -a $LOG_FILE
echo "Reboot $NEXT_REBOOT of $REBOOT_COUNT" | tee -a $LOG_FILE

# Check recovery flag
RECOVERY=$(cat /home/chronos/Local\ State 2>/dev/null | grep -o '"EnrollmentRecoveryRequired":true')
echo "Recovery flag = ${RECOVERY:-false}" | tee -a $LOG_FILE

# Record timestamp
date | tee -a $LOG_FILE

# Update state for next run
echo $NEXT_REBOOT > $STATE_FILE

if [ $NEXT_REBOOT -lt $REBOOT_COUNT ]; then
  echo "Rebooting in 5 seconds..." | tee -a $LOG_FILE
  echo "Re-run this script after reboot to continue testing" | tee -a $LOG_FILE
  sleep 5
  sudo reboot
else
  echo "Test complete after $REBOOT_COUNT reboots" | tee -a $LOG_FILE
  rm -f $STATE_FILE
fi
```

### Test 3: Log Analysis Script

```bash
#!/bin/bash
# Analyze chrome logs for DM token check patterns

echo "=== DM Token Check Analysis ==="

# Count DM token check failures
FAIL_COUNT=$(sudo grep -c "no DM token" /var/log/chrome/chrome 2>/dev/null)
echo "DM Token check failures: $FAIL_COUNT"

# Show recent failures with context
if [ $FAIL_COUNT -gt 0 ]; then
  echo ""
  echo "Recent failures:"
  sudo grep -B5 -A5 "no DM token" /var/log/chrome/chrome 2>/dev/null | tail -50
fi

# Check enrollment recovery flag
RECOVERY=$(cat /home/chronos/Local\ State 2>/dev/null | grep -o '"EnrollmentRecoveryRequired":[^,}]*')
echo ""
echo "Recovery flag: ${RECOVERY:-not set}"

# Check UMA metrics
echo ""
echo "Checking UMA metrics..."
sudo grep "Enterprise.EnrolledPolicyHasDMToken" /var/log/chrome/chrome 2>/dev/null | tail -20

# Summary
echo ""
echo "=== Summary ==="
if [ $FAIL_COUNT -gt 0 ] || [ ! -z "$(echo $RECOVERY | grep true)" ]; then
  echo "⚠️ POTENTIAL BUG DETECTED"
  echo "Recommendation: Update to ChromeOS version with fix"
else
  echo "✅ No issues detected"
fi
```

## Interpreting Results

### ✅ Device NOT Affected (Good)
- Enrollment recovery flag never set on properly enrolled device
- No "no DM token" errors in logs
- Device boots normally after reboots
- `chrome://policy` shows all policies correctly
- UMA histogram shows only true values for DM token checks

### ❌ Device IS Affected (Bug Present)
- Enrollment recovery flag set (`EnrollmentRecoveryRequired:true`)
- "no DM token" errors appear in chrome logs
- Device occasionally shows enrollment screen after reboot
- Policy fetches fail intermittently
- UMA histogram shows false values for DM token checks

### ⚠️ Inconclusive
- Intermittent issues that may or may not be this bug
- Device has other enrollment/policy problems
- Recommend: Test on a known-good network connection
- Recommend: Test with a freshly enrolled device

## Recommended Testing Matrix

| Device Type | Version | Test Priority | Test Scenario |
|-------------|---------|---------------|---------------|
| Old hardware (3+ years) | Pre-fix | HIGH | Scenario 1 + 2 |
| New hardware (<1 year) | Pre-fix | MEDIUM | Scenario 1 |
| All devices | Post-fix | LOW | Quick verification |
| High-value devices | Pre-fix | HIGH | All scenarios |

## Reporting Results

When reporting test results, include:

1. **Device Information**:
   - Model number
   - ChromeOS version
   - Hardware specs (CPU, RAM, storage type)

2. **Test Results**:
   - Which tests were run
   - Number of reboots tested
   - How many times bug triggered
   - Frequency of false positives

3. **Logs** (if possible):
   - Relevant lines from `/var/log/chrome/chrome`
   - Contents of enrollment recovery flag
   - Output from test scripts

4. **Impact Assessment**:
   - Number of devices in fleet
   - How many potentially affected
   - Business impact of false enrollments

## Mitigation Until Fix

If you detect the bug on your devices:

### Short-term
1. **Document affected devices** and models
2. **Reduce unnecessary reboots** where possible
3. **Monitor** for enrollment recovery triggers
4. **Have re-enrollment process** ready

### Medium-term
1. **Plan ChromeOS update** to version with fix
2. **Test fix in lab** before fleet rollout
3. **Prioritize update** for most-affected device models

### Long-term
1. **Monitor metrics** to ensure fix is effective
2. **Document lessons learned** for future issues
3. **Improve monitoring** for enrollment health

## Support and Questions

- For technical details: See `docs/enterprise/unenrollment_bug_fix.md`
- For ChromeOS version information: Check release notes
- For enterprise support: Contact your ChromeOS support channel

## Quick Reference Card

```
┌─────────────────────────────────────────────────┐
│ QUICK BUG DETECTION                             │
├─────────────────────────────────────────────────┤
│ 1. Check chrome://policy                       │
│    - Should show device policies               │
│                                                 │
│ 2. Reboot enrolled device                      │
│    - Should go to login screen                 │
│    - NOT to enrollment screen                  │
│                                                 │
│ 3. Check logs (if root access):                │
│    grep "no DM token" /var/log/chrome/chrome   │
│    - Should find 0 matches                     │
│                                                 │
│ 4. Check recovery flag (if root access):       │
│    cat /home/chronos/Local\ State              │
│    - Should NOT have EnrollmentRecoveryRequired │
│                                                 │
│ ✅ All pass = Not affected                     │
│ ❌ Any fail = Likely affected                  │
└─────────────────────────────────────────────────┘
```
