# Lab Walkthrough: Username Enumeration via Different Responses

**Author:** RAT Bully
**Platform:** PortSwigger Web Security Academy
**Category:** Authentication Vulnerabilities
**Difficulty:** Apprentice

---

## Objective

Identify valid usernames on a login form by spotting subtle differences in the server's error responses, then use that information as the foundation for a credential attack.

## Methodology

This walkthrough follows a 3-track approach: **Dev Tools** (reconnaissance), **Burp Suite** (attack execution), and **Concept** (the underlying vulnerability logic). Learning all three in parallel builds recognition skills that transfer across tools and applications, not just familiarity with one lab.

### 1. Dev Tools — Recon

Before touching Burp, the login page's HTML was inspected via view-source (`Ctrl+U`) to check for hidden form fields  specifically a CSRF token that could complicate automation later. None were present in this lab instance, which simplified the attack setup considerably (no need to dynamically refresh a token between requests).

A baseline login attempt was also submitted with the browser's Network tab open, to observe:
- **Request method:** `POST`
- **Field names:** `username`, `password`
- **Default error response:** the message returned for a failed login attempt, used as a baseline for comparison

### 2. Burp Suite — Attack Execution

The baseline request was captured in **Proxy > HTTP History** and sent to **Intruder**. Since only the username needed to vary, the **Sniper** attack type was used:

- Payload position marked on the `username` field
- Candidate username wordlist loaded as the payload set
- Password field left as a fixed placeholder value (irrelevant at this stage)

The attack was run, and the **Length** column in the Intruder results grid was used to spot outliers most invalid usernames return an identically-sized response, while a valid username (paired with any wrong password) triggers a different message length.

### 3. Concept — Why It Works

The vulnerability exists because the application returns different error messages depending on whether the *username* is invalid versus whether the username is valid but the *password* is wrong. Comparing response length/content across all Intruder results reveals which response(s) diverged from the majority pattern  exposing valid usernames without ever needing to guess a correct password.

## Result

A valid username was successfully identified by isolating the response that differed from the standard "invalid username" pattern, confirming the account exists on the system.

## Key Takeaways

- Inconsistent error messaging between "bad username" and "bad password" is a common but high-impact authentication flaw.
- Automated recon (Burp Intruder + response length as a signal) is far faster and more reliable than manually testing each candidate by hand.
- This finding is a natural precursor to a full credential/brute-force attack  once a username is confirmed valid, only the password remains unknown.

## Next Steps

Apply this same 3-track methodology to the **"Broken brute-force protection, IP block"** lab, using the enumerated username as the target for a Pitchfork-based password brute-force attack that alternates target guesses with a known-good login to reset the lockout counter.
