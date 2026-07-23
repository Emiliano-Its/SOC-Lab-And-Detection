# Security Incident Summary — For Leadership

**Date of incident:** June 29, 2026, 9:05 PM
**System affected:** One employee workstation (finance department test machine)
**Prepared by:** Jose Emiliano Cortez Rivera, SOC Analyst
**Audience:** Executive / non-technical leadership

---

## What happened

Our monitoring system detected suspicious activity on an employee workstation: a script ran
that downloaded a program and saved it to the computer's hard drive, disguised in a way
commonly used to hide malicious software from a normal user. This is a well-known pattern
attackers use to get a foothold on a machine before stealing data or spreading further into
the network.

Our security tools flagged this within the same second it happened. Two automated alerts
fired back-to-back: one for the suspicious file being saved to disk, and one for the script
that saved it. There was no delay between the activity happening and our team being notified.

## What we found

The activity was consistent with an attacker trying to establish a way to quietly send data
out of the company network — commonly called "exfiltration." At the time of the alert, we
had visibility into the fact that a file was dropped and a script ran it, but we did **not**
have confirmation that data actually left the network. We are treating this conservatively,
as if it could have led to data loss, because that is the safer assumption for a security
team to make.

## What could have been exposed

This machine did not have access to customer financial records or production systems — it
was a limited-scope endpoint. Based on what we can see, no customer data, payment
information, or other sensitive company data was confirmed to have left the network. We are
continuing to review logs to close out this question with full confidence.

## What we did about it

- **Isolated the affected computer** from the rest of the network immediately, so it could
  not be used to reach any other system.
- **Stopped the suspicious program** and removed the file it had downloaded.
- **Reset the passwords** for any accounts that were logged into that machine at the time,
  as a precaution.
- **Confirmed the computer was clean** before reconnecting it to the network.

None of these steps required taking any other system offline, and there was no disruption to
normal business operations outside of that one workstation.

## What this means for the business

No evidence of customer or financial data loss has been found. The bigger takeaway is that
our monitoring worked as intended — it caught this within seconds, on the first machine it
touched, before it could spread. That said, this incident showed us a gap: we can currently
see when a suspicious file is dropped and run, but we do not yet have a rule that confirms
whether that program successfully "phoned home" to an outside server. We are closing that gap
next.

## What we're doing next

- Adding a network-level alert that specifically catches this type of program trying to
  contact an outside server, so we can say with certainty whether data left the network —
  not just that suspicious activity occurred.
- Reviewing whether this type of file-drop behavior should automatically lock the affected
  computer out of the network the moment it's detected, rather than requiring a person to do
  it manually.
- No changes to customer-facing systems or services are needed as a result of this incident.
