# Cal app: 3-D Secure "safe transaction" approval is biometric-only and records a silent decline

**Filed:** 2026-11-19
**Status:** confirmed (traced with adb logcat + UI dumps on a real purchase; approval succeeded only after the fingerprint worked)
**Vendor:** Cal, Israel Credit Cards Ltd (`com.onoapps.cal4u` 6.26.20, Android 16)
**Full report:** https://github.com/danielrosehill/Cal-Safe-Transaction-Approval-Defect
**History:** same symptom reported to Cal customer service July 2026, unresolved.

## Summary

Online purchases on a Cal card send an SMS asking for approval in the app. The deep link works
and an approval screen exists, but it accepts only a fingerprint. When the fingerprint is unavailable or fails
(screen protector, app biometric key invalidated after a password change), it shows an error, closes within
2-3 s and Cal records a **decline**. The app then shows the transactions list with no message, so the
cardholder sees "nothing to approve" while the merchant says "declined in the Cal app".

## Workaround

Make sure Cal's fingerprint login works *before* paying (remove privacy protectors over an
in-display sensor), tap the SMS immediately, approve, continue to biometric, scan. Never close the
approval screen.

## Disclosure

Sent to: pending (privacy@icc.co.il DPO and the Public Complaints Commissioner form are the
confirmed non-service channels; Cal has no security.txt or disclosure programme).
