# Quick Reference: Testing for Chromebook Unenrollment Vulnerability

## 🚨 Critical Test for System Administrators

Use this guide to quickly test if your Chromebooks are vulnerable to unauthorized unenrollment.

---

## ⚡ 5-Minute Quick Test

### Test Device: Latest Chrome OS Version

**Steps:**
1. ✓ Enroll a test Chromebook in your organization
2. ✓ Verify it shows "Managed" in device settings (chrome://policy)
3. ✓ Attempt to enter Developer Mode:
   - Turn off Chromebook
   - Press: **Esc + Refresh + Power**
   - At recovery screen, press: **Ctrl + D**
4. ✓ Observe the result

**Expected Results:**
- ✅ **PROTECTED**: "Developer mode is disabled by your administrator"
- ❌ **VULNERABLE**: Device allows entering developer mode

---

## 🔍 Detailed Test for Older Versions

### Test Device: Chrome OS Version < M122

**Steps:**
1. ✓ Enter Recovery Mode:
   - Turn off device
   - Press: **Esc + Refresh + Power**
2. ✓ Insert Chrome OS recovery USB
3. ✓ Complete recovery process
4. ✓ After recovery, observe enrollment screen

**Expected Results:**
- ✅ **PROTECTED**: Device forces re-enrollment (cannot skip)
- ❌ **VULNERABLE**: Option to skip enrollment present

---

## 🛡️ If Devices are VULNERABLE

### Immediate Actions (< 1 hour):

```
☑️ 1. Enable Forced Re-enrollment
   - Google Admin Console → Devices → Chrome
   - Device Settings → Enrollment & Access
   - ✓ Force device to re-enroll on powerwash

☑️ 2. Block Developer Mode
   - Device Settings → System Settings
   - ✓ Block developer mode (DeviceBlockDevmode)

☑️ 3. Set Up Alerts
   - Google Admin Console → Reports → Alerts
   - ✓ Alert on device unenrollment
   - ✓ Alert on policy fetch failures
```

### Short-term Actions (< 1 week):

```
☑️ 4. Update Chrome OS
   - Schedule update to latest version
   - Priority: Devices physically with students

☑️ 5. Audit Current Devices
   - Check for unexpectedly unenrolled devices
   - Investigate recent enrollment status changes

☑️ 6. Physical Security
   - Consider asset tags
   - Implement check-in/check-out procedures
```

---

## 📊 Monitoring Checklist

### Daily:
- [ ] Check Admin Console for unenrolled devices
- [ ] Review enrollment status change alerts

### Weekly:
- [ ] Review device policy compliance reports
- [ ] Check for devices with repeated policy fetch errors

### Monthly:
- [ ] Audit complete device enrollment list
- [ ] Review security incident reports
- [ ] Update device inventory

---

## 🔔 Signs of Exploitation

**Indicators a device may have been tampered with:**

⚠️ Device shows as "Not enrolled" unexpectedly
⚠️ Device enrollment status changes without admin action  
⚠️ Device has recent developer mode access in logs
⚠️ Multiple enrollment/unenrollment cycles
⚠️ Policy fetch errors followed by unenrollment
⚠️ Missing DM token errors in device logs

**Action:** Investigate immediately, re-enroll device, check for pattern across device fleet.

---

## 💡 Quick Commands for IT Staff

### Check Device Status (on device):
```
chrome://policy           # View current policies
chrome://device-log       # Check device logs
crossystem                # Check firmware settings
```

### Admin Console Quick Links:
```
Devices → Chrome → Devices               # All devices
Reports → Device Info                    # Device details
Reports → Alerts                         # Set up alerts
Settings → Device Settings               # Configure policies
```

---

## 📞 Get Help

**For Urgent Security Issues:**
1. Check device logs for tampering indicators
2. Review `chromebook_unenrollment_vulnerability.md` for details
3. Contact Chrome Enterprise Support
4. Follow your organization's incident response procedures

**For Testing Questions:**
- See `chromebook_unenrollment_vulnerability.md` Section: Testing Instructions
- Review test scenarios 1-5 for detailed steps

**For Implementation Help:**
- See `chromebook_unenrollment_fix_summary.md`
- Review recommended policies section

---

## ✅ Verification After Applying Fix

**After updating devices or applying policies:**

1. **Test Protection:**
   - Attempt developer mode: Should be blocked ✓
   - Attempt recovery mode re-enrollment skip: Should force enrollment ✓

2. **Verify Policies:**
   - Check `chrome://policy` shows DeviceBlockDevmode = true
   - Check forced re-enrollment enabled in Admin Console

3. **Monitor Metrics:**
   - No unexpected unenrollments in 24 hours ✓
   - Policy fetch errors resolved ✓

---

## 🎯 Success Criteria

Your devices are protected when:

✅ Developer mode is blocked on all managed devices
✅ Forced re-enrollment is enabled
✅ All devices on latest Chrome OS (or update scheduled)
✅ Monitoring alerts are configured
✅ No unexpected unenrollments in past 7 days
✅ Regular audit schedule established

---

**Print this page and keep at IT help desk for quick reference!**

Last Updated: 2025-10-16
