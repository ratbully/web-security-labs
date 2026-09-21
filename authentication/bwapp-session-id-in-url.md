# bWAPP: Session ID Exposure in URL (Session Hijacking)

**Author:** RatBully
**Platform:** bWAPP (Broken Web Application)
**Lab/Module:** Session Management - Session ID in URL
**Category:** Broken Authentication / Session Management
**Levels Covered:** Low, Medium, High

---

## Objective

This report continues the ongoing authentication vulnerability learning journey, shifting focus from credential disclosure (bWAPP, DVWA) to **session management** - specifically, whether a valid session token can be exposed and reused by an attacker without ever knowing the victim's username or password. Session hijacking is one of the more serious findings in this category, since it results in full account takeover rather than just information disclosure.

## Methodology

This assessment combined dev tools and manual browser manipulation rather than Burp Suite, since the exploitation technique here relies on directly editing a browser's stored cookie value rather than modifying an in-flight request:

- **Dev Tools** - identifying where the session ID appears (URL vs. cookie) across all three security levels
- **Manual session hijacking** - copying a live session ID from one browser and manually injecting it into a second, unauthenticated browser
- **Concept** - understanding why exposing a session token outside the cookie mechanism creates a hijacking risk

---

## Low Security: Session ID Exposed in URL

**Finding:** After logging in, the session identifier (`PHPSESSID`) was displayed directly in the browser's address bar as a URL parameter, rather than being confined to the cookie alone.

**Example observed:**
```
http://192.168.56.103/bWAPP/smgmt_sessionid_url.php?PHPSESSID=5fc85ssld35bsn5m8na2j97lj3
```

**Exploitation process:**
1. Logged into bWAPP as `bee` in Browser A and copied the live `PHPSESSID` value from the URL.
2. Created a separate test user, and opened bWAPP in a completely separate browser (Browser B), which had never authenticated.
3. Used browser DevTools to manually edit Browser B's `PHPSESSID` cookie, replacing its value with the session ID copied from Browser A.
4. Reloaded the authenticated portal page in Browser B.

**Result:** Browser B was granted full authenticated access as the original session's user, without ever submitting a username or password. This confirms a working **session hijacking** exploit - an attacker who obtains this URL (via browser history, a shared link, server logs, or a leaked `Referer` header) could fully take over the victim's session.

**Root cause:** Exposing the session token in the URL creates multiple leak paths that a cookie-only implementation avoids - browser history, external referrer headers, and server/proxy access logs are not designed to protect sensitive values the way cookies are.

---

## Medium and High Security: Properly Confined to Cookie

**Finding:** At both Medium and High security levels, the session ID no longer appeared anywhere in the URL.

**Verification:** Checked via DevTools (Application/Storage -> Cookies) and the Network tab across both levels - the session token was present only in the `Cookie` header of each request, with no trace in the URL, hidden form fields, or elsewhere in the page.

**Result:** Both levels correctly confine the session token to the cookie mechanism, closing off the URL-based exposure paths identified at Low.

---

## Comparative Summary

| Level | Session ID Location | Hijackable via URL Leak? |
|---|---|---|
| Low | URL parameter + cookie | Yes - token fully exposed in address bar |
| Medium | Cookie only | No - no URL exposure |
| High | Cookie only | No - no URL exposure |

## Key Takeaways

- A session token's *location* matters as much as its randomness. Even a well-generated session ID becomes a liability if it's exposed somewhere insecure, like a URL.
- Session hijacking doesn't require breaking encryption or guessing anything - if the token itself can be obtained, it can simply be replayed in another browser to gain full access.
- Cookies exist as the standard mechanism for session tokens specifically because browsers handle them differently from URLs - they're excluded from `Referer` headers by default and not retained in visible browsing history the way a URL is.
- This finding pairs naturally with the earlier "Strong Sessions" module, which addresses a related but distinct concern: even when a session ID stays properly inside a cookie, that cookie still needs the right protections (`HttpOnly`, `Secure`) to resist theft through other attack paths like XSS or network sniffing.

## Tools Used

- **Browser Dev Tools** - inspecting and manually editing cookie values across two separate browser sessions
- **Manual cross-browser testing** - used in place of Burp Suite to directly demonstrate real-world session hijacking mechanics
