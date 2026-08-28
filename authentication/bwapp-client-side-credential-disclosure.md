# bWAPP: Client-Side Credential Disclosure Across Security Levels

**Author:** RatBully
**Platform:** bWAPP (Broken Web Application)
**Category:** Broken Authentication / Client-Side Information Disclosure
**Levels Covered:** Low, Medium, High

---

## Objective

This report is part of an ongoing authentication vulnerability learning journey spanning multiple platforms (PortSwigger, DVWA, bWAPP), each chosen to reinforce the same core concepts from a different angle rather than in isolation. Where the PortSwigger labs focused on server-side signals (response differences, broken lockout logic) uncovered through Burp Suite, this bWAPP series shifts the lens to **client-side vulnerabilities**  cases where the flaw isn't in how the server responds, but in what the browser is trusted to keep secret.

The objective here is twofold: assess how the same login form handles credential protection as bWAPP's security level increases, and demonstrate that "security" implemented only on the client side  no matter how it's disguised  is not real protection. Together with the earlier labs, this builds a fuller picture of authentication weaknesses across both the server and client sides of an application.

## Methodology

As with previous labs, this assessment follows a 3-track approach:
- **Dev Tools** — primary technique used across all three levels; view-source and manual page inspection
- **Manual Analysis** — tracing obfuscated logic by hand (Medium level)
- **Concept** — understanding *why* each mistake is a vulnerability, not just how to exploit it

Unlike the brute-force and enumeration labs, **Burp Suite was not required** here  each level's credentials were disclosed directly in client-side content, making automated attack unnecessary. This itself is a notable finding: attackers don't need heavy tooling when developers leave secrets exposed in the front end.

---

## Low Security: Hidden Field Disclosure

**Finding:** The password was present in plaintext, sitting inside a hidden HTML input field in the page source.

**How it was found:** Viewing the page source (`Ctrl+U`) revealed a hidden `<input>` element with the password value directly readable  no encoding, no obfuscation, just plain text left visible to anyone who checked the source.

**Root cause:** A basic developer oversight  likely a leftover debug/test value that was never removed before the form was considered "complete." This is one of the most common real-world findings in code review: developers store sensitive values client-side for convenience during development and forget to strip them out.

**Severity note:** Trivial to exploit  requires no tools beyond a browser.

---

## Medium Security: Obfuscated Password via JavaScript

**Finding:** The password was not directly visible, but reconstructable by tracing JavaScript logic embedded in the page source.

**How it was found:** The source contained a base string (`"bash update killed my shells!"`) and a series of single-letter variables, each assigned a character from that string using `.charAt(index)`. Several variable names were **reused multiple times**, meaning only each variable's *final* assignment mattered (JavaScript overwrites earlier `var` declarations).

**Process:**
1. Indexed the base string from position 0
2. Traced each variable (`d`, `j`, `k`, `q`, `x`, `t`, `o`, `g`, `h`, `p`) to its last declared `.charAt()` value
3. Concatenated the results in the order defined by the `secret` variable

**Result:** `hulk smash!`

**Root cause:** This is a slightly more sophisticated attempt at "security through obscurity"  scattering and reusing variable names to make manual tracing harder. However, since all the logic still executes client-side and is fully visible in the page source, it provides no real protection against a patient reviewer. This is a good example of obfuscation being mistaken for encryption or hashing  no cryptographic function was involved at any point.

---

## High Security: Credential Hint in Plain Text

**Finding:** Rather than hiding the password in code, the page displayed a plain-text riddle directly on the login page itself:

> Remember: *a bee is a bug...*

**How it was found:** No source inspection was even necessary  the hint was visible in the rendered page content.

**Interpretation:** The phrase directly maps to bWAPP's default account structure  username `bee`, password `bug`. What appears to be a "harder" security level actually discloses the credentials more directly than Medium did, just phrased as a riddle instead of a raw value.

**Root cause:** A misguided attempt to make credential disclosure feel like a security feature ("test your skills") rather than removing it entirely. This highlights an important lesson: increasing apparent complexity does not equal increasing security  if anything, this level was the fastest to solve.

---

## Comparative Summary

| Level | Disclosure Method | Effort to Exploit | Tools Needed |
|---|---|---|---|
| Low | Plaintext hidden field | Trivial | Browser (view-source) |
| Medium | Obfuscated JS variable logic | Low–Moderate | Browser + manual tracing |
| High | Plain-text hint phrase | Trivial | None (visible on page) |

## Key Takeaways

- Client-side "protection" — whether hidden fields, scrambled variables, or riddles — is not authentication security. Anything sent to the browser can be read by the user.
- Obfuscation is not encryption. Scrambling logic makes analysis slower, not impossible, and gives false confidence to developers who mistake it for real protection.
- Difficulty level does not always correlate with actual security strength — this series showed that "High" was arguably the easiest to solve.
- Real fixes require moving all credential validation and secret handling server-side, with nothing sensitive ever reaching client-rendered code.

## Tools Used

- **Browser Dev Tools** — view-source and manual page inspection across all three levels
- **Manual analysis** — tracing JavaScript variable reassignment to reconstruct the Medium-level password
- **CyberChef** *(optional/verification only)* — used to visually index character positions in the base string; no decoding or hash-cracking was actually required, since no hashing was present at any level
