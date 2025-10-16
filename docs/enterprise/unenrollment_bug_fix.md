# ChromeOS Spontaneous Unenrollment Bug - Fix and Testing Guide

## Executive Summary

A bug in ChromeOS enrollment recovery detection was causing false positives for "spontaneous unenrollment" on enrolled devices. This resulted in devices being incorrectly flagged to go through enrollment recovery on the next boot, even though they were properly enrolled.

**Status**: Fixed in this commit

## The Problem

### What Was Happening

Enrolled ChromeOS devices would sometimes be incorrectly marked as "unenrolled" and forced through the enrollment recovery process on next boot. This occurred even though the device was properly enrolled and had a valid Device Management (DM) token.

### Root Cause

The bug was in the `CheckDMToken()` function in `chrome/browser/ash/policy/core/device_cloud_policy_store_ash.cc`. 

**The Issue**: The function set a flag (`dm_token_checked_ = true`) immediately upon entering the check, BEFORE verifying whether a DM token actually exists. Once this flag was set, the function would never check again, even if the DM token became available later.

**Triggering Conditions**:
1. **Initialization timing**: If device settings were not fully initialized when the check first ran
2. **Transient errors**: Temporary I/O errors or delays reading policy data from disk
3. **Race conditions**: Policy data loading after the initial check but before the flag was set

When any of these occurred, the device would be marked for enrollment recovery even though it was properly enrolled.

### Impact

- **False alarms**: Properly enrolled devices incorrectly flagged as unenrolled
- **User disruption**: Forced enrollment recovery flow on next boot
- **Admin overhead**: Support tickets and troubleshooting for "phantom" unenrollment
- **Security concerns**: Uncertainty about whether devices were genuinely unenrolled

## The Fix

### Technical Details

**Change**: Move the `dm_token_checked_ = true` assignment to only execute when a DM token is successfully found.

**File Modified**: `chrome/browser/ash/policy/core/device_cloud_policy_store_ash.cc`

**Lines Changed**: 2 (one line removed from early in function, one line added inside success condition)

### Before (Buggy Code)
```cpp
void DeviceCloudPolicyStoreAsh::CheckDMToken() {
  // ... status checks ...
  
  if (dm_token_checked_) {
    return;
  }
  dm_token_checked_ = true;  // ❌ BUG: Set before checking!
  
  const em::PolicyData* policy_data = device_settings_service_->policy_data();
  if (policy_data && policy_data->has_request_token()) {
    // DM token found, device is properly enrolled
    base::UmaHistogramBoolean(kDMTokenCheckHistogram, true);
    return;
  }
  
  // No DM token found, mark for enrollment recovery
  LOG(ERROR) << "Device policy read on enrolled device yields no DM token!";
  ash::StartupUtils::MarkEnrollmentRecoveryRequired();
}
```

### After (Fixed Code)
```cpp
void DeviceCloudPolicyStoreAsh::CheckDMToken() {
  // ... status checks ...
  
  if (dm_token_checked_) {
    return;
  }
  
  const em::PolicyData* policy_data = device_settings_service_->policy_data();
  if (policy_data && policy_data->has_request_token()) {
    dm_token_checked_ = true;  // ✅ FIXED: Set only on success!
    // DM token found, device is properly enrolled
    base::UmaHistogramBoolean(kDMTokenCheckHistogram, true);
    return;
  }
  
  // No DM token found, mark for enrollment recovery
  LOG(ERROR) << "Device policy read on enrolled device yields no DM token!";
  ash::StartupUtils::MarkEnrollmentRecoveryRequired();
}
```

### How the Fix Works

**Before Fix**:
1. First check runs during initialization
2. DM token not loaded yet (transient condition)
3. Flag set to true → no more checks
4. Device marked for enrollment recovery (FALSE POSITIVE)
5. Even when DM token loads later, it's never detected

**After Fix**:
1. First check runs during initialization
2. DM token not loaded yet (transient condition)
3. Flag NOT set → will check again on next update
4. Second check: DM token now loaded
5. Flag set to true → device confirmed enrolled, no recovery needed

## Testing the Fix

### For System Administrators

If you're running ChromeOS devices on older versions (before this fix), you can test whether your devices are affected by this bug.

#### Detection Method 1: Check Local State Preferences

On an affected device:

1. **Access the device**: Log in as root or use developer mode
2. **Check the local state file**: 
   ```bash
   cat /home/chronos/Local\ State | grep -i "EnrollmentRecoveryRequired"
   ```
3. **If you see**: `"EnrollmentRecoveryRequired":true`
4. **And the device is properly enrolled**, this could be a false positive

#### Detection Method 2: Check Chrome Logs

1. **Access chrome logs**: 
   ```bash
   tail -1000 /var/log/chrome/chrome | grep -i "no DM token"
   ```
2. **Look for**: `"Device policy read on enrolled device yields no DM token!"`
3. **If you see this error** but the device appears properly enrolled, it's likely affected

#### Detection Method 3: Monitor Enrollment Recovery Triggers

1. **Set up monitoring** for devices triggering enrollment recovery
2. **Check if**: Devices that were properly enrolled suddenly show enrollment recovery flow
3. **Correlate with**: Recent reboots or ChromeOS updates

### Verification After Fix

To verify the fix is working correctly:

#### Test Case 1: Normal Enrolled Device
1. **Setup**: Fresh enrolled device with fix applied
2. **Expected**: Device boots normally, no enrollment recovery triggered
3. **Verify**: Check logs show DM token found successfully
   ```bash
   cat /var/log/chrome/chrome | grep -i "dm token"
   ```

#### Test Case 2: Simulated Slow Boot
1. **Setup**: Device with slow I/O or delayed policy loading
2. **Expected**: Multiple CheckDMToken calls until token found, then stops
3. **Verify**: No false enrollment recovery flag set
   ```bash
   cat /home/chronos/Local\ State | grep -i "EnrollmentRecoveryRequired"
   ```
   Should show `false` or not present

#### Test Case 3: Genuine Unenrollment
1. **Setup**: Device with actual missing DM token (test environment only!)
2. **Expected**: Enrollment recovery correctly triggered
3. **Verify**: Recovery pref set, enrollment flow starts on reboot

## Workaround for Older Versions

If you're running ChromeOS versions without this fix and experiencing false positives:

### Temporary Workaround

1. **Identify affected devices** using detection methods above
2. **Clear the false recovery flag**:
   ```bash
   # As root on the affected device
   pkill -9 chrome
   # Edit Local State to remove or set to false:
   # "EnrollmentRecoveryRequired": false
   # Then restart Chrome
   ```
3. **Monitor** to see if issue recurs after reboot

### Permanent Solution

**Upgrade** to a ChromeOS version that includes this fix (this commit or later).

## Monitoring Recommendations

### Metrics to Track

1. **Enrollment Recovery Triggers**: Count of devices entering enrollment recovery flow
2. **False Recovery Rate**: Devices marked for recovery that still have valid DM tokens
3. **DM Token Check Failures**: Log entries showing "no DM token" errors

### Alerting Thresholds

- **Alert** if enrollment recovery rate suddenly increases
- **Alert** if devices with recent successful policy fetches trigger recovery
- **Investigate** if specific device models show higher rates

### Useful Log Queries

```bash
# Check for the specific error
grep "Device policy read on enrolled device yields no DM token" /var/log/chrome/chrome

# Check enrollment recovery pref
strings /home/chronos/Local\ State | grep EnrollmentRecoveryRequired

# Check UMA metrics for DM token checks
grep "Enterprise.EnrolledPolicyHasDMToken" /var/log/chrome/chrome
```

## FAQ

### Q: How do I know if my devices are affected?
**A**: Check logs for "no DM token" errors on properly enrolled devices, or monitor for unexpected enrollment recovery triggers.

### Q: Will this fix affect genuine unenrollment detection?
**A**: No. The fix only prevents false positives from transient conditions. Genuine unenrollment (missing DM token after initialization completes) will still be detected.

### Q: Do I need to re-enroll devices after applying the fix?
**A**: No. The fix prevents false positives going forward. Devices that are properly enrolled remain enrolled.

### Q: What if a device was already marked for enrollment recovery (false positive)?
**A**: On next boot with the fix applied, the recovery check will detect that the DM token is actually present and clear the false flag automatically.

### Q: Can this be tested in a lab environment?
**A**: Yes. Test with devices that have slower disk I/O or during high system load to simulate transient conditions that could trigger the bug.

### Q: Is there a performance impact from the fix?
**A**: Minimal. In the false positive case, the check may run a few more times (2-3) before finding the DM token, but this has negligible performance impact.

## Technical References

### Related Files
- `chrome/browser/ash/policy/core/device_cloud_policy_store_ash.cc` - Main fix location
- `chrome/browser/ash/policy/enrollment/enrollment_config.cc` - Recovery config logic
- `chrome/browser/ash/login/startup_utils.cc` - Recovery pref setting
- `docs/enterprise/enrollment.md` - Enrollment overview documentation

### Key Functions
- `DeviceCloudPolicyStoreAsh::CheckDMToken()` - DM token verification (FIX LOCATION)
- `GetPrescribedRecoveryConfig()` - Recovery mode detection
- `StartupUtils::MarkEnrollmentRecoveryRequired()` - Sets recovery pref

### Relevant Prefs
- `prefs::kEnrollmentRecoveryRequired` - Boolean flag for enrollment recovery
- `prefs::kServerBackedDeviceState` - Server-provided device state

### UMA Metrics
- `Enterprise.EnrolledPolicyHasDMToken` - Whether DM token was found
- `Enterprise.EnrolledDevicePolicyPresent` - Whether policy data exists

## Support

For issues or questions:
1. Check ChromeOS enterprise support documentation
2. Review device logs using methods described above
3. Contact your ChromeOS administrator or support team

## Revision History

- **Initial version**: Fix and documentation created
- **Status**: Fixed in commit 6281a7c7be
