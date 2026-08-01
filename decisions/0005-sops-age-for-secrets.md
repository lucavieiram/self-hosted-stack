# 5. SOPS and age: secrets encrypted inside the config repo

**Status:** accepted, in production
**Date:** 2026-07

## Context

The infrastructure repo holds compose files, the reverse proxy config and the backup
script. Those need an API token for DNS-01, a database root password, and a restic
repository password. Two bad options present themselves immediately: commit them in
plaintext, or keep them out of the repo entirely and rely on remembering to copy the
right file to the right host during a rebuild.

The second is how disaster recovery quietly fails. The repo restores perfectly and the
stack will not start, because the one file that was never in it is gone with the host.

## Decision

SOPS with age. Secrets are encrypted at rest and committed; the private key lives only on
the host and in a password manager.

```
secrets/*
!secrets/*.enc
!secrets/README.md
```

Encrypted `.enc` files are committed, everything else in that directory is ignored. A
rebuild is: clone the repo, drop in the age key, decrypt. Config and its secrets travel
together and cannot drift apart, because they are in the same commit.

age rather than GPG because the key is a single line, there is no keyring, no trust model,
no expiry, and no subkey confusion. For one operator and one host, GPG's flexibility is
entirely cost.

## Rejected

**A secrets manager (Vault, Infisical).** The correct answer at team scale. Here it is
another service to run, back up, and unseal — and it introduces a circular dependency,
because the thing holding my secrets needs to be up before the stack that needs them can
start.

**Cloud KMS.** Removes the "don't lose the key" problem by moving custody to a provider.
Rejected because it makes the whole stack unrecoverable without that account, and because
owning key custody is exactly the fundamental I want to actually understand.

**Environment files kept out of git, copied by hand.** What I was doing. It works right
up until the rebuild, which is the only moment it matters.

## Cost

The age private key is now a single point of failure: lose it and every `.enc` file is
unrecoverable. It lives in a password manager and in an offline copy. Key rotation is
also manual — re-encrypt every file against the new recipient — so it happens rarely,
which is its own mild risk.

## Gotcha worth writing down

`sops` searches for `.sops.yaml` by walking *upward from the directory of the file being
encrypted*, not from the working directory. With the config at the repo root and the
secrets in an unrelated absolute path, no rule matched — and it produced empty output
files rather than an error. Passing the recipient explicitly with `--age` avoids relying
on discovery at all.

Dotenv files additionally need `--input-type dotenv --output-type dotenv`, or the
round-trip silently reshapes them into something the container cannot read.

Both failures were silent. Anything that writes secrets should be verified by decrypting
the result and diffing it against the original, never by checking the exit code.
