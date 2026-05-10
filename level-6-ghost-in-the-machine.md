# BreachLab, Ghost Track Level 6: Ghost in the Machine

**Status:** Solved
**Tools allowed:** `env`, `echo`

---

## The Theme

Credential extraction. Environment variables are how secrets leak into process lists, crash logs, and CI pipelines every single day. This level is built around that exact failure mode.

---

## Recon

First move after getting shell access on any box is to see what the environment is already telling you. The `env` command prints every variable the current shell has loaded. User info, paths, runtime config, and anything the system (or a sloppy admin) planted in memory.

```bash
ghost6@breachlab:~$ env
```

The output is a wall of text. Most of it is boilerplate that exists on every Linux box and can be skipped. Things like `SHELL`, `HOME`, `LANG`, `PWD`, `LOGNAME`, `TERM`, `PATH`, `LS_COLORS`, `HISTSIZE`, and `SSH_CONNECTION`.

Filtering those out leaves the variables that someone deliberately set on this machine. Three of them stood out:

```
API_DIGEST=M252X0wzNGtzXzN2M3J5dGgxbmc=
TRACE_SALT=bW9uaXRvcmluZ19rZXlfZGVsdGE3
RUNTIME_TOKEN=c3lzdGVtX3Rva2VuX2dhbW1hX3Yz
```

Three tells these are Base64:

1. The character set is only `A-Z`, `a-z`, `0-9`, and `+/=`.
2. `API_DIGEST` ends in `=`, which is Base64 padding.
3. The variable names sound like the kind of infrastructure config you'd skim past on a real production server, which is exactly the trap.

---

## Reasoning Over Guessing

With three suspicious Base64 strings, the temptation is to decode all of them and look for whatever looks like a password. That works, but it's not analyst thinking. The better move is to read the variable names first and ask which one is actually a credential by definition.

**DIGEST** is a hash or fingerprint. One-way. You don't authenticate with a digest, you compare against one.

**SALT** is a random value mixed into a hash. Not secret by itself, not a credential.

**TOKEN** is a string you present to prove identity to a system. That's a credential by definition.

`RUNTIME_TOKEN` was the right first decode.

---

## The Decode

Since the variable is already in the environment, no need to retype the Base64 string. Just reference it by name.

```bash
ghost6@breachlab:~$ echo "$RUNTIME_TOKEN" | base64 -d
system_token_gamma_v3
```

That's the level 7 password.

### What the command is doing

`echo "$RUNTIME_TOKEN"` expands the variable to its value and prints it to stdout. The `|` pipes that output into the next command. `base64 -d` runs the base64 utility in decode mode, turning the encoded string back into the original bytes.

Other ways to do the same thing:

```bash
base64 -d <<< "$RUNTIME_TOKEN"            # here-string, no echo process
printf '%s' "$RUNTIME_TOKEN" | base64 -d  # no trailing newline
```

---

## Don't Stop at the First Hit

The password got me into level 7, but the level flag was still hiding. CTF designers plant decoys alongside the real prize, and real attackers harvest the whole environment because they don't know which variable holds what until they look.

Decoded the other two:

```bash
ghost6@breachlab:~$ echo "$TRACE_SALT" | base64 -d
monitoring_key_delta7

ghost6@breachlab:~$ echo "$API_DIGEST" | base64 -d
3nv_L34ks_3v3ryth1ng
```

`TRACE_SALT` was the red herring. `API_DIGEST` held the actual flag.

The lesson here is that a single successful decode doesn't mean recon is done. If three variables look suspicious, decode all three.

---

## Why This Matters in the Real World

Base64 is encoding, not encryption. It's reversible by anyone with a terminal. No key, no crack, no skill required. Anywhere it shows up pretending to protect a secret, it's a costume, not security.

Real-world examples where this exact pattern leaks credentials:

* `.env` files accidentally committed to a public Git repo
* Docker `ENV` directives baked into image layers
* Kubernetes ConfigMaps where someone confused them with Secrets
* CI/CD runners (GitHub Actions, GitLab CI, Jenkins) dumping environment into build logs when a job crashes
* `ps auxe` output exposing another user's environment on a shared host
* Sentry or Datadog crash reports capturing the full env at the moment of failure
* JWT tokens, which are just three Base64 sections separated by dots. Anyone can decode the header and payload without the signing key

Any time a value looks "technical enough" to be a secret but uses only `A-Z a-z 0-9 + / =`, run it through `base64 -d`. The 33% size increase and the trailing `=` padding are the giveaways.

---

## What I Learned

### About the `base64` command

`base64` is a Linux utility whose only job is converting data between two formats. Raw bytes on one side, and a safe text alphabet of `A-Z`, `a-z`, `0-9`, `+`, `/`, with `=` padding on the other side.

By default it encodes. The `-d` flag flips it to decode.

```bash
echo "hello" | base64           # encode:  aGVsbG8K
echo "aGVsbG8K" | base64 -d     # decode:  hello
```

Base64 exists because a lot of internet plumbing was originally built for plain English text. Email, HTTP headers, JSON, XML, URLs. Raw binary data breaks those channels. Null bytes cut strings short, newlines confuse parsers, weird characters get mangled in transit. Base64 takes any bytes at all and re-expresses them using only 64 safe characters so the data survives. The tradeoff is the encoded version is about 33% larger than the original.

The security takeaway is that because Base64 looks technical, people instinctively treat it like it's encrypted. It isn't. It's reversible by anyone, no key required. In a security review, "we Base64-encoded the password" is a red flag every single time.

Flags worth keeping in the toolkit:

```bash
base64 file.txt              # encode a file
base64 -d encoded.txt        # decode a file
echo "..." | base64 -d       # decode a string via pipe
base64 -d <<< "..."          # decode a string via here-string
base64 -w 0                  # encode without line wrapping
```

### About environment variables

`env` shows everything in the current shell's memory. On a real box, that includes anything an admin or a deployment script set, which is often more than they meant to share. Variables ending in `_TOKEN`, `_KEY`, `_SECRET`, `_DIGEST`, `_SALT`, or `_PASSWORD` are worth a second look. So is anything that uses only Base64 characters.

You can reference any variable by name with `$VARNAME` instead of retyping its value. That's the move once you've spotted it in `env` output.

### About reasoning over guessing

The biggest takeaway from this level had nothing to do with commands. It was the reflex to read variable names like an analyst before touching them. A token is a credential. A digest is a fingerprint. A salt is an ingredient. Knowing the difference saved me from decoding in the wrong order and probably saved time on harder boxes later.

The second piece of that same habit is to not stop after the first win. The password felt like the answer. It wasn't. The flag was sitting in a different variable the whole time, and skipping the rest of the recon would have cost me the points.

### About the pipe and the shell

`echo`, `cat`, `printf`, here-strings (`<<<`), and pipes (`|`) all do the same basic job from different angles. They move text from one place to another. `base64 -d` doesn't care who hands it data, only that data shows up on its standard input. Once that clicked, I stopped memorizing commands and started thinking in data flow. That's the Unix philosophy in one line: small tools that do one thing, connected together.

---

## Takeaway

The whole level fits in one habit. When you land on a box, run `env`, ignore the boilerplate, decode anything that looks like Base64, and don't stop after the first hit. That's it. That's the entire technique behind a real category of production breaches.
