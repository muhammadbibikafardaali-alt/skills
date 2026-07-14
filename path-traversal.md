# Path traversal

Study notes from the PortSwigger Web Security Academy - Practitioner path.
Also called directory traversal or "dot-dot-slash." The bug lets an
attacker read (and sometimes write) files outside the directory the app
meant to expose.

## the mental model

Anywhere an app takes a filename or path from user input and passes it to
the filesystem, this is possible. The classic case is a URL parameter that
names an image, PDF, or template:

    GET /image?file=cat.png

If the server effectively does `open("images/" + file)`, then supplying:

    file=../../../../etc/passwd

resolves to /etc/passwd and the response body is its contents.

Windows target for the same idea:

    C:\Windows\win.ini

## defenses and their bypasses

The Practitioner labs walk you up a ladder of "developer tries to fix it,
here's how you get past that fix."

### 1. no defense

Straight ../ traversal works.

    file=../../../etc/passwd

### 2. the app strips ../ once, non-recursively

Nest the sequence so that stripping one occurrence still leaves one behind:

    file=....//....//....//etc/passwd

Trace it: `....//` has `../` sitting inside it (chars 3,4,5). Strip that,
what's left is `../`. This only works because the strip runs once, not
in a loop.

### 3. the app blocks the literal string ../

URL-encode the traversal so it doesn't match the string filter but does
decode back to ../ on the server:

    file=%2e%2e%2fetc%2fpasswd

Double-encode when the app decodes once and then checks - so after one
decode you have %2e%2e%2f, which passes the "no ../" check, and a second
decode inside the framework turns it into ../:

    file=%252e%252e%252fetc%252fpasswd

Some stacks also accept non-standard encodings (overlong UTF-8,
backslashes on Windows) that get past filters written only for ../.

### 4. the app requires input to start with the expected base path

Server-side check: "input must start with /var/www/images". Include the
expected prefix and then traverse out from there:

    file=/var/www/images/../../../etc/passwd

Path resolution collapses ../ regardless of what came before, so the
check passes and the read still lands on /etc/passwd.

### 5. the app requires input to end with a specific extension

Server-side check: "input must end in .png". Classic bypass on older
runtimes is the null byte:

    file=../../../etc/passwd%00.png

The underlying C-style string handling in older PHP or Java treats %00
as end-of-string when opening the file, but the extension check saw
".png" at the end of the string and passed. This is historical - modern
PHP/Java don't have this - but still shows up in labs and on old apps.

## finding it in practice

signals in a target app:

- any parameter that looks like a filename, path, template name,
  or document ID
- download / export / image / include / view endpoints
- error messages that leak filesystem paths ("could not find
  /var/www/uploads/foo.png")
- responses whose status, length, or content-type changes when you
  supply a different filename

fuzzing with Burp Intruder:

- payload list: ../, ....//, %2e%2e%2f, %252e%252e%252f, absolute paths
- target files:
    /etc/passwd, /etc/hostname, /proc/self/environ, /proc/self/cmdline,
    C:\Windows\win.ini
- watch for changes in status code, response length, content-type

## impact

- read-only traversal: source code disclosure, credentials in config
  files, private keys, /etc/passwd, cloud instance metadata mounts,
  session files
- traversal in a write context (upload paths, log paths, template paths)
  frequently becomes RCE - drop a webshell into a served directory,
  overwrite a cron file, poison a template

## defensive takeaways

- don't take file paths from user input at all if you can help it.
  map an opaque ID server-side to a known filename.
- if you must accept a path: canonicalise first (realpath /
  File.getCanonicalPath / Path.resolve), then verify the resolved path
  is inside the intended base directory. reject anything else.
- do that check AFTER decoding and normalisation, not before.
- run the app as an OS user without read access to sensitive files.
- containerise or chroot the file-serving process where feasible.

## references

- PortSwigger Web Security Academy - Path traversal
- OWASP - Path Traversal
- CWE-22: Improper Limitation of a Pathname to a Restricted Directory

## notes to self

- keep confusing "strip ../ once" with "block ../ literally" - they need
  different bypasses. `....//` for the strip, encoding for the block.
- null byte bypass is historical, useful for context, won't work on
  current stacks. don't waste time on it against a modern target.
- most lab time is spent identifying WHICH defense the app uses, not
  applying the bypass. read the app first, then pick the payload.
