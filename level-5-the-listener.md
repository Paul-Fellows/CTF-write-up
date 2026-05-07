# BreachLab -- Ghost Track: Level 5
**The Listener · 200 pts**

---

## What This Level Is About

The story behind this level is that a character named KAEL stopped trusting files after a
security incident. Instead of writing credentials to disk, he ran a service that serves the
password live over the network. His thinking was that if it is not on disk, it cannot be
found. He forgot about the ports.

The lesson here is real: sensitive data does not have to be in a file to be exposed. If a
service is listening on a port and handing out credentials, anyone who finds that port can
grab them. This is exactly what penetration testers look for during network reconnaissance.

---

## Getting In

Logged into the ghost5 machine using the password from Level 4:

```bash
ssh ghost5@204.168.229.209 -p 2222
```

---

## Looking Around

First thing I did was check the hosts file to understand what machines were on the network:

```bash
cat /etc/hosts
```

This showed two IPs -- `172.19.0.2` (my machine) and `172.19.0.1` (the gateway). I spent
a while scanning the gateway looking for the listener, which was the wrong move. The service
was running locally on ghost5 the whole time.

---

## Finding the Listener

Ran a full port scan on localhost to see what was actually running on this machine:

```bash
nmap -p 1-65535 127.0.0.1
```

This came back with a bunch of open ports in the 30000 range, including several marked as
"unknown." Those are the interesting ones. Port 30100 was the first one I hit:

```bash
nc 127.0.0.1 30100
```

The service introduced itself immediately:

```
GHOST PROTOCOL -- CHANNEL A
________________________________

This channel is informational only.

Authentication token: GHOST
Secure channel: port 30101

Send the token to receive your credential.
```

Port 30100 was an info channel. It told me exactly what to do next -- connect to port 30101
and send the token GHOST.

---

## Getting the Credential

Connected to the secure channel:

```bash
nc 127.0.0.1 30101
```

It prompted for authentication:

```
AUTHENTICATE:
```

Sent the token:

```
GHOST
```

And it handed over the credential:

```
Credential: P0rts_N3v3r_L13
```

---

## Flag

```
P0rts_N3v3r_L13
```

---

## What I Learned

The biggest mistake I made on this level was assuming the listener was on the external host
at `172.19.0.1`. I spent a lot of time scanning that IP when the answer was sitting on
localhost the whole time. The takeaway is to always scan your own machine, not just the
network around you.

The two-port setup was a good design. Port 30100 was public and informational -- it told you
the token and pointed you to the real channel. Port 30101 was the authenticated channel that
only gave up the credential if you sent the right token. This mirrors how real services work:
a discovery endpoint and a secured endpoint behind it.

Banner grabbing with nc is simple but powerful. You connect, read what the service says, and
respond accordingly. No exploits, no guessing -- just listening to what the service tells you.

---

*Solved on BreachLab · Ghost Track · [breachlab.org](https://breachlab.org)*
