# bWAPP: Strong Sessions (Cookie Security Flags)

**Author:** RatBully
**Platform:** bWAPP (Broken Web Application)
**Lab/Module:** Session Management - Strong Sessions
**Category:** Broken Authentication / Session Management
**Levels Covered:** Low, Medium, High

---

## Objective

This report is a companion piece to the earlier "Session ID in URL" finding, addressing a related but distinct session management concern: even when a session token is properly confined to a cookie, that cookie still needs the right protective flags to resist theft through other attack paths, such as script-based access (XSS) or interception over unencrypted connections.

## Methodology

- **Source code review (via SSH)** - read the PHP logic behind this module directly to understand exactly which cookie flags each security level is intended to set
- **Dev Tools** - verified actual cookie behavior in the browser (Application/Storage -> Cookies) against what the source code predicted

## Findings by Level (Explained Simply)

To understand this lab, it helps to think of a session cookie like a **house key**. Once you log in, the website hands your browser a "key" (the cookie) that proves you're allowed back in without logging in again on every page. The real question this lab tests is: **how well is that key protected from being stolen?**

Two separate protections matter here, and they guard against two *different* kinds of theft:

- **HttpOnly** :— stops a thief who has slipped a malicious script onto the page from simply reading the key straight out of the browser (this is the "no one can pick the key up off the counter" protection).
- **Secure** :— stops the key from ever being handed over on an unsafe, unencrypted line of communication, like a phone call anyone could be listening in on (this is the "the key is only ever handed over in a locked room" protection).

A truly strong session needs *both* protections. Missing either one leaves a different door open for an attacker.

---

**Low  No key protection exists at all**

At this level, the page doesn't even bother generating a protected cookie. There's nothing to steal because there's nothing being handed out securely in the first place  it's the equivalent of the website never even bothering to hand you a key, so there's no lock to evaluate here.

---

**Medium  Half the protection is in place**

At Medium, a cookie called `top_security_nossl` is created, and it does have the **HttpOnly** flag turned on. In our house-key analogy: **the key can no longer be picked up directly by a malicious script running on the page**  a real, meaningful improvement.

However, the **Secure** flag is missing. That means the "key" is still allowed to be handed over on an unencrypted line, like the website is willing to shout your key out loud over an open phone line, where anyone nearby with the right equipment (packet sniffing on shared WiFi, for example) could simply listen in and grab it as it goes by.

So Medium fixes *one* of the two doors, but leaves the other wide open.

**What was actually observed:**
- `HttpOnly: true` ✅
- `Secure: false` ❌

---

**High  Fully protected in theory, but the lock never gets installed in this environment**

Reading the actual source code, High is *written* to do everything right: it creates a cookie called `top_security_ssl` with **both** protections turned on HttpOnly *and* Secure. In theory, this is the fully secure version: the key can't be grabbed by a script, and it can never be handed over except through a properly locked, encrypted line.

But here's the twist: when this was actually tested in the browser, **no cookie showed up at all** not a broken one, not a partial one, literally nothing. At first glance that might look like the security level is doing *worse* than Medium. It isn't.

**Why nothing appeared, explained simply:**
Modern browsers have their own built-in rule: they will **flat-out refuse to accept any cookie marked "Secure" unless the entire website connection itself is already running over a secure, encrypted channel (HTTPS)**. This lab environment is running on plain, unencrypted HTTP, there's no "locked room" available at all for the browser to use.

So think of it like this: bWAPP correctly tried to hand over a key sealed in a locked box, **but insisted the handoff itself happen inside a secure room that doesn't exist in this building**. The browser, quite correctly, refuses to complete a handoff it can't guarantee is safe — so it throws the whole attempt away rather than doing it insecurely. Nothing arriving is actually the browser protecting you, not a failure.

**What was actually observed:**
- No cookie appeared in DevTools at all
- Source code confirms it *should* have `HttpOnly: true` and `Secure: true`, if HTTPS were available



## Comparative Summary

| Level | Cookie | HttpOnly | Secure | Real-World Risk |
|---|---|---|---|---|
| Low | None generated | - | - | No session cookie to evaluate |
| Medium | `top_security_nossl` | Yes | No | Vulnerable to network interception (no TLS enforcement) |
| High | None observed (blocked by browser) | Yes (per source) | Yes (per source) | Not observable in this HTTP-only environment - browser blocks Secure cookies over HTTP by design |

## Key Takeaways

- Session cookie security depends on more than just where the token lives (URL vs. cookie, as shown in the companion report) - the cookie's own flags determine what kinds of theft it resists.
- `HttpOnly` defends against client-side script attacks (XSS); `Secure` defends against network-level interception. They address different threats and both matter.
- A properly implemented `Secure` flag will cause modern browsers to refuse to store the cookie at all outside of HTTPS  meaning a security feature can sometimes look like "nothing happened" in testing, when in fact it's actively doing its job.
- This finding highlights the importance of testing security-relevant behavior in an environment that matches real-world conditions (HTTPS) - some protections are only fully observable, and fully meaningful, once TLS is actually in place.

## Tools Used

- **SSH + source code review** :- read `smgmt_strong_sessions.php` directly to determine intended cookie behavior per level
- **Browser Dev Tools** :- inspected actual cookie attributes (HttpOnly/Secure) to confirm or explain behavior observed at each level
