# Solutions

Exploits for the **WebAuthn 1** and **WebAuthn 2** challenges of
`CyberstepsDE/WebAuthn-Challenge` (`server.py`).

```
solutions/
├── bash/WebAuthn1/webauthn1_exploit.sh      # curl + jq + openssl + xxd
├── bash/WebAuthn2/webauthn2_exploit.sh      # curl + jq + openssl + xxd
├── php/WebAuthn1/webauthn1_exploit.php      # ext-openssl + ext-curl
├── php/WebAuthn2/webauthn2_exploit.php      # ext-openssl + ext-curl
├── python/WebAuthn1/webauthn1_exploit.py    # cryptography + stdlib urllib
└── python/WebAuthn2/webauthn2_exploit.py    # cryptography + stdlib urllib
```

All three implementations are self-contained software authenticators: they
speak the raw WebAuthn wire format (CBOR attestation object, COSE EC2 public
key, DER-encoded ECDSA signature) without any WebAuthn library.

## Running

```bash
# 1. start the challenge
docker compose up -d --build webauthn

# 2. python (only dependency: cryptography)
python3 -m venv .venv && .venv/bin/pip install cryptography
.venv/bin/python solutions/python/WebAuthn2/webauthn2_exploit.py

# 3. bash
bash solutions/bash/WebAuthn2/webauthn2_exploit.sh

# 4. php
php solutions/php/WebAuthn2/webauthn2_exploit.php
```

The target can be changed with `WEBAUTHN_BASE`, `WEBAUTHN_ORIGIN`,
`WEBAUTHN_RP_ID` and `WEBAUTHN_USERNAME`.

## WebAuthn 1 — "can you log in?"

The challenge exposes only an API, so the browser part has to be recreated by
hand:

1. `GET /api/register` → RP ID, challenge, `pubKeyCredParams`.
2. Generate an EC P-256 keypair and build a `fmt: "none"` attestation object:
   `authenticatorData = rpIdHash(32) || flags(1) || signCount(4) || aaguid(16)
   || credIdLen(2) || credId || COSE_Key`, flags `0x45` (UP | UV | AT), plus
   `clientDataJSON = {"type":"webauthn.create","challenge":…,"origin":…}`.
3. `POST /api/register` with `username` + `reg`.
4. `GET /api/login` → challenge, then build an assertion:
   `authenticatorData = rpIdHash || flags(1) || signCount(4)`, flags `0x05`
   (UP | UV — the server sets `require_user_verification=True`), sign
   `authenticatorData || sha256(clientDataJSON)` with ECDSA/SHA-256 and send it
   as `POST /api/login` with `userHandle = <our username>`.

Result: `flag1` for our own account.

## WebAuthn 2 — "can you hack it?"

Goal: log in as the pre-seeded `admin` account.

### The bugs

**1. The credential ID that is *checked* is not the one that is *stored*.**

`server.py:156` rejects a registration if the client-supplied ID is taken:

```python
if username in users or base64url_to_bytes(reg['id']).hex() in credentials:
    return make_response("Username or ID already exists", 400)
```

but the credential is stored under the ID that comes from the *attested
credential data* inside `authenticatorData` (`server.py:54`):

```python
'cred_id': verified.credential_id.hex(),
```

`py_webauthn` only enforces `id == rawId`; it never compares them against the
credential ID inside the attestation statement. So the two IDs can differ.

**2. Admin's credential ID is public.**

`server.py:172-173` puts the stored credential ID of a user straight into the
login options:

```python
creds = [PublicKeyCredentialDescriptor(id=bytes.fromhex(users[username]))]
```

`GET /api/login?username=admin` therefore leaks the credential ID that is also
the key of the `credentials` dict entry holding admin's public key.

**3. The account is selected by an unsigned, client-supplied field.**

`server.py:198` takes the username from the assertion's `userHandle`, which is
*not* covered by the assertion signature (only `authenticatorData` and
`clientDataHash` are signed):

```python
username = base64url_to_bytes(auth['response']['userHandle']).decode()
```

and `server.py:69` then picks the public key to verify against from that
username:

```python
cred = credentials[users[username]]
```

**4. Bonus: the challenge check is a no-op.**

Both ceremonies compute the "expected" challenge from the client-supplied
`clientDataJSON` (`server.py:45` and `server.py:73`), so the challenge is never
really validated. (The RP ID and the origin *are* checked, but both are public
knowledge.)

### The attack

1. **Leak** admin's credential ID:
   `GET /api/login?username=admin` → `allowCredentials[0].id`.
2. **Poison** the `credentials` entry with our own public key: register a
   credential whose attested credential ID is *admin's* credential ID but whose
   COSE public key is *ours*. The top-level `id`/`rawId` is a random decoy, so
   the uniqueness check at `server.py:156` passes, while `save_data()`
   (`server.py:95-103`) overwrites `credentials[<admin credential id>]` with
   our public key — the account `admin` now points at a key we control.
3. **Log in** as admin: `POST /api/login` with `userHandle = "admin"` and an
   assertion signed by our key. The server resolves the public key via
   `credentials[users["admin"]]`, which is now *our* key, so the signature
   verifies.

   The `id`/`rawId` of the assertion is only used for sign-count bookkeeping in
   `update_data()` (`server.py:106-110`, `credentials[cred_id]['sign'] = …`),
   so it has to be a credential ID that already exists in `credentials`,
   otherwise the request dies with a `KeyError` (HTTP 500) instead of returning
   the flag. Admin's credential ID is used for that.

Result: `flag2`.

### Why the "obvious" attack does not work

Simply re-sending our own validly signed assertion with `userHandle` swapped
to `"admin"` fails with HTTP 401: the server verifies the signature against
admin's *real* hardcoded public key, and the `userHandle` is not part of the
signed data. The credential-ID confusion above is what makes the swap
meaningful: it lets us replace the key that admin's account is verified
against.
