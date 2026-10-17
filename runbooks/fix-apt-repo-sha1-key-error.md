---
tags:
  - runbook
  - "#apt"
---

# Fix an apt repo failing on a SHA1 key signature

Fixes `apt update` failing on a third-party repo with an `sqv` error like "Signing key ... is not bound" and "SHA1 is not considered secure." `sqv` has rejected SHA1 signatures since 2026-02-01, so a repo key with a SHA1 binding signature fails even though the rest of apt is fine. The fix is a fresh copy of the vendor's key in its own keyring file, in place of the key embedded in the repo file. It only works if the vendor has republished the key with a signature `sqv` accepts. Elastic's 8.x repo worked this way on 2026-10-16, on the Kali Purple VM.

## Step 1: Find the repo file
Look at its `Signed-By:` field. Mine had the public key embedded inline.
```
grep -rl <vendor> /etc/apt/sources.list /etc/apt/sources.list.d/
```

## Step 2: Back it up
Keep the backup outside the `sources.list.d` folder.
```
sudo cp /etc/apt/sources.list.d/<file> ~/<file>.bak
```

## Step 3: Download a fresh key and make a keyring
```
curl -fsSL <vendor-key-url> -o /tmp/<vendor>.key
sudo gpg --dearmor --yes -o /usr/share/keyrings/<vendor>-keyring.gpg /tmp/<vendor>.key
```

## Step 4: Point the repo file at the keyring
Delete the whole inline key block under `Signed-By:`, through the `END PGP PUBLIC KEY BLOCK` line, and replace it with one line. Leave the other fields alone.
```
Signed-By: /usr/share/keyrings/<vendor>-keyring.gpg
```

## Step 5: Update
```
sudo apt update
```

## Verification
`apt update` finishes without an error for that repo. If the same error comes back, the vendor has not fixed their key. Two fallbacks:
- Rename the repo file to end in `.disabled`. It costs nothing unless the vendor's packages are needed.
- Extend `sha1.second_preimage_resistance` in `/etc/crypto-policies/back-ends/apt-sequoia.config`, copied from `/usr/share/apt/default-sequoia.config`. I have not tried this, and it loosens the policy for every repo.

I would not use `[trusted=yes]`, since that turns off signature checking for the repo.

## Related Notes
- [Take a baseline of a fresh VM](take-a-vm-baseline.md)

## Tags
#runbook
