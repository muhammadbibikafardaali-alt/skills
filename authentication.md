# Authentication vulnerabilities

Study notes from the PortSwigger Web Security Academy - Practitioner path.
Covers the common ways login, MFA, and password reset flows break, plus
the mitigations. Point of these notes is to remember the bug classes, not
the specific labs.

## the mental model

Authentication answers "who are you." Anywhere the app asks that question -
login, password reset, MFA challenge, "remember me" cookie, account recovery -
is attack surface. The bugs tend to fall into a few families:

- leaks that reveal whether a user exists
- letting an attacker try too many guesses
- trusting client-supplied state
- second-factor steps that can be skipped or replayed

Almost everything below is a specific case of one of those.

## password-based login

### username enumeration

The app tells you (deliberately or by accident) which usernames exist.
That turns generic brute-force into targeted brute-force.

signals:

- different error messages ("invalid username" vs "invalid password")
- different HTTP status or response length between valid and invalid users
- different response time - valid users take longer because the app
  actually hashes a real password; invalid users short-circuit
- account-lockout messages that only appear for real accounts

workflow:

- send a wordlist of usernames with a fixed dummy password through Burp
  Intruder
- sort by response length, then by response time
- outliers are your valid users

### brute-force basics

Once you have valid usernames, you brute-force the password field. On
Practitioner labs the point isn't "brute-force with no protection" - it's
brute-force against something that looks protected but isn't.

### flawed brute-force protection

things that look like protection but aren't:

- IP-based lockout - bypass with X-Forwarded-For, rotating the value each
  request. The app trusts the header and thinks each request is a new IP.
- per-account lockout after N failures - bypass with password spraying.
  One password across many accounts stays under the per-account counter.
- lockout counter that resets on any successful login - log in as your
  own account between attempts to reset the victim's counter.

### keeping-me-logged-in cookies

"Remember me" cookies are often a serialised or hashed version of
username + password. Classic lab format is base64 of
`username:md5(password)`. If you can grab the cookie (XSS, network,
whatever) and the hash is weak, offline cracking gives you the password
itself - not just session access.

## multi-factor authentication

### 2FA simple bypass

The app enforces 2FA only through the URL flow, not server-side session
state. After submitting valid credentials, browse directly to /account or
/my-account instead of /login2. The server never actually checked whether
the 2FA step completed.

### 2FA broken logic

The 2FA verification request carries the username in a hidden field or a
cookie. Change it to a different valid username - the app looks up that
user's 2FA code and validates against it, not against the actual session
user. If you can also brute-force the code (see below), you're in.

### 2FA code brute-force

If the code is 4-6 digits and the verification endpoint has no rate limit
or lockout, brute-force the whole code space with Intruder. Watch for
session invalidation after too many attempts - some apps drop the session
even without an explicit "locked" message, which just looks like every
guess being wrong.

## password reset

### token leaked via Host header

The reset link is built from the Host header of the reset-request:

    https://{host}/reset?token=abc123

If the app trusts Host, an attacker submits a reset request with
Host: attacker.com. The reset email goes to the victim but contains a
link pointing at attacker.com carrying the victim's real token. The
victim clicks - the token lands on the attacker's server (via the
request itself, or via Referer on a linked resource).

### token leaked via Referer

The reset page loads an external resource (image, tracking pixel, script).
The reset token, sitting in the page URL, leaks out through the Referer
header on that outbound request.

### broken reset logic

Practitioner labs also cover:

- reset endpoint accepts a `username` parameter that decides whose
  password gets changed. Swap it to the victim's username.
- reset token isn't tied to a specific user - request a token as
  yourself, then use that token to reset someone else's password.

## defensive takeaways

Because this is what matters in the day job:

- consistent error messages AND consistent response times for valid vs
  invalid usernames. timing matters as much as the string.
- rate limits at both the account level and the IP level; captcha
  after N failures.
- lockouts that don't leak whether an account exists.
- MFA enforced server-side as a mandatory step of session establishment,
  not as a page in a flow the user can skip past.
- password reset tokens: random, single-use, short-lived, tied to a
  specific user, delivered on URLs built from server-side config -
  never from request headers.
- Host header validation / allowlist for outgoing links and any URL
  the app itself generates.

## references

- PortSwigger Web Security Academy - Authentication vulnerabilities
- OWASP ASVS chapter 2 (Authentication)
- OWASP Top 10 A07:2021 - Identification and Authentication Failures

## notes to self

- do each lab manually before reaching for Intruder - the point is to
  see the request pattern, not to click "start attack"
- write a small python script for at least one bug per family. that
  cements what the raw request looks like, not just "the Burp workflow"
- password spraying and username enumeration together are the most
  common real-world entry point. worth extra reps.
