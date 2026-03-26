# Case #001 — Suspicious Account Creation
**Date:** 2026-03-26 
**Analyst:** ellianima 
**Agent:** Strawberry-Plump-Jelly-Silly-Fun (192.168.1.7)
**Severity:** Medium 
**Verdict:** True Positive (analyst simulation)

## Summary
Wazuh Rule 60109 (Level 8) triggered on Windows 11 endpoint.
A new local user account "MIZISUA" was created via net user command.

## MITRE ATT&CK
- Tactic: Persistence
- Technique: T1098 — Account Manipulation

## Timeline
| Time | Event |
|---|---|
| ~17:00 | net user MIZISUA /add executed |
| ~17:01 | Wazuh Rule 60109 triggered |
| 21:00 | Alert triaged by analyst |

## Investigation
- Command detected: `net user MIZISUA /add`
- User context: Administrator
- Account verified as analyst-created test account

## Other Alerts Triaged
| Rule | MITRE | Level | Verdict |
|---|---|---|---|
| 60106 | T1078 | 3 | False Positive |
| 60170 | T1484 | 5 | False Positive |
| 504/506 | T1562.001 | 3 | False Positive |

## Action Taken
Test account removed. No escalation required.

## Lessons Learned
Wazuh detected account creation within seconds and correctly
mapped to MITRE T1098. False positives from normal Windows
background processes confirmed as benign.

'''
Notes for tomorrow: blablablabla analyse moreeee, keep consistent tabs, always.
'''
