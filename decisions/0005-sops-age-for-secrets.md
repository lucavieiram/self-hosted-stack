# 5. SOPS + age, secrets encrypted in the repo

**Accepted, in production. 2026-07.**

## Context

The infra repo holds compose files, proxy config and the backup script. Those need a DNS
API token, a DB root password and the restic repo password.

Two obvious options, both bad: commit them plaintext, or keep them out of the repo and
remember to copy the right file to the right host on rebuild. The second is how DR
quietly fails — the repo restores perfectly and the stack won't start.

## Decision

SOPS with age. Encrypted at rest, committed. Private key lives on the host and in a
password manager.

```
secrets/*
!secrets/*.enc
!secrets/README.md
```

Rebuild is: clone, drop in the age key, decrypt. Config and secrets are in the same
commit and can't drift apart.

age over GPG because it's one line, no keyring, no trust model, no expiry, no subkeys. For
one operator GPG's flexibility is entirely cost.

## Rejected

- **Vault / Infisical.** Correct at team scale. Another service to run, back up and
  unseal, and it's circular — the thing holding my secrets must be up before the stack
  that needs them.
- **Cloud KMS.** Makes the stack unrecoverable without that account, and moves key custody
  away from the fundamental I want to understand.
- **Env files copied by hand.** What I was doing. Works until the rebuild, which is the
  only moment it matters.

## Cost

Lose the age key and every `.enc` is unrecoverable. It's in a password manager and an
offline copy. Rotation is manual — re-encrypt every file — so it happens rarely.

## Gotchas

`sops` finds `.sops.yaml` by walking upward from **the directory of the file being
encrypted**, not the working directory. Mine was at the repo root, the secrets were at an
unrelated absolute path, no rule matched — and it wrote empty files instead of erroring.
Pass `--age` explicitly and skip discovery.

Dotenv needs `--input-type dotenv --output-type dotenv` or the round-trip silently
reshapes it into something the container can't read.

Both failures were silent with exit 0. Verify by decrypting and diffing against the
original.
