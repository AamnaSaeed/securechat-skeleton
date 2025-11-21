SecureChat – Assignment #2 (CS-3002 Information Security, Fall 2025)

This repository contains my complete implementation of a console-based, PKI-enabled Secure Chat System in Python.
The project demonstrates how cryptographic primitives combine to achieve:

Confidentiality • Integrity • Authenticity • Non-Repudiation (CIANR)

All security is implemented explicitly at the application layer, not via TLS/SSL wrappers.

🧩 Overview

The original repository provided a project skeleton with empty modules and TODO comments.
I implemented:

A complete application-layer secure protocol

PKI-based certificate verification

Diffie–Hellman key exchanges

AES-128 encryption for message confidentiality

RSA SHA-256 signatures for message integrity & authenticity

Replay protection using message sequence numbers

Append-only transcript logging

Signed session receipts enabling non-repudiation

Both client and server operate over plain TCP sockets, with cryptography built purely in Python.

🏗️ Folder Structure (with Implementation Details)
securechat-skeleton/
├─ app/
│  ├─ client.py              # Client workflow: hello, DH, login/register,
│  │                         # signed DH session, encrypted chat, receipts
│  ├─ server.py              # Server workflow mirroring client logic
│  ├─ crypto/
│  │  ├─ aes.py              # AES-128 ECB + PKCS7 + Base64 wrapper
│  │  ├─ dh.py               # Classic modular DH + shared key derivation
│  │  ├─ pki.py              # X.509 load, CA validation, CN check, fingerprints
│  │  └─ sign.py             # RSA sign/verify (SHA-256, PKCS#1 v1.5)
│  ├─ common/
│  │  ├─ protocol.py         # JSON encode/decode, message models
│  │  └─ utils.py            # sha256_hex, timestamps, base64 helpers
│  └─ storage/
│     ├─ db.py               # MySQL: salted SHA-256 password storage
│     └─ transcript.py       # Append-only transcript + transcript hashing
├─ scripts/
│  ├─ gen_ca.py              # Create Root CA (RSA + self-signed X.509)
│  └─ gen_cert.py            # Generate server/client certificates signed by CA
├─ tests/manual/NOTES.md     # Manual test checklist and Wireshark instructions
├─ certs/                    # Certificates + keys (gitignored)
├─ transcripts/              # Session logs (gitignored)
├─ .env.example              # DB + certificate configuration
├─ .gitignore                # Ignore certs, transcripts, venv, .env
├─ requirements.txt          # Python dependencies
└─ .github/workflows/ci.yml  # Build sanity test

⚙️ Setup Instructions
1. Clone & environment setup
git clone <your-fork-url>
cd securechat-skeleton
python -m venv .venv
source .venv/Scripts/activate   # Git Bash on Windows
pip install -r requirements.txt
cp .env.example .env

2. Start MySQL (via Docker recommended)
docker run -d --name securechat-db \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=securechat \
  -e MYSQL_USER=scuser \
  -e MYSQL_PASSWORD=scpass \
  -p 3306:3306 mysql:8

3. Initialize database schema
python -m app.storage.db --init


Creates table:

users(email, username, salt, pwd_hash)


with salted SHA-256 password storage.

4. Generate PKI (Root CA + server/client certificates)
python scripts/gen_ca.py --name "FAST-NU Root CA"
python scripts/gen_cert.py --cn server.local --out certs/server
python scripts/gen_cert.py --cn client.local --out certs/client


Certificates produced:

ca.key.pem, ca.cert.pem

server.key.pem, server.cert.pem

client.key.pem, client.cert.pem

5. Run the system

Server:

python -m app.server


Client:

python -m app.client

🔒 Detailed Implementation Summary
1. Application-Layer Protocol

Messages are transmitted as:

4-byte length header + JSON payload


ensuring reliable framing over TCP.

Implemented message types:

hello

dh_client, dh_server

register, login

dh_session, dh_session_server

msg (encrypted chat message)

receipt (signed session receipt)

2. PKI & Certificate Validation

pki.py performs:

Loading X.509 certificates

CA signature verification

Validity period checks

Common Name (CN) validation

Computing SHA-256 fingerprint for receipts

Extracting RSA public key

Invalid/self-signed certificates produce BAD_CERT.

3. Diffie–Hellman Exchanges
Phase 1 — Temporary DH

Used to encrypt:

Registration

Login

Derives temporary AES key.

Phase 2 — Signed Session DH

Used for actual chat encryption.

Client signs A
Server signs B
Mutually authenticated key exchange ensures:

authenticity

protection against man-in-the-middle

fresh session key

Session key =

SHA256(shared_secret)[0:16]


→ AES-128 key.

4. AES Encryption Layer

Implemented in aes.py:

AES-128 in ECB mode

PKCS#7 padding

Base64 output for JSON transport

Functions:

encryptEcbB64(key16, plaintext)
decryptEcbB64(key16, ciphertext_b64)


Used for login/registration + all chat messages.

5. Digital Signatures for Messages

For every chat message:

Compute hash over canonical message:

seqno || timestamp || ciphertext


Sign SHA-256 digest with RSA PKCS#1 v1.5

Receiver recomputes digest and verifies signature

If altered → SIG_FAIL.

6. Replay Protection

Each message includes a monotonically increasing seqno.

If a message arrives with:

seqno <= last_seqno


the receiver prints:

REPLAY


and discards message.

7. Transcript + Non-Repudiation

Every message is logged:

seqno | timestamp | ct | sig | fingerprint


At session end:

Hash entire transcript using SHA-256

Sign this digest with private RSA key

Both peers exchange signed receipts

Each peer verifies:

fingerprint match

digest match

valid RSA signature

This establishes non-repudiation.

Receipt files stored in:

server_receipts/
client_receipts/

🧪 Test Evidence Checklist (Verified)

✔ Wireshark: only encrypted AES ciphertext visible
✔ BAD_CERT: invalid/self-signed cert rejected
✔ SIG_FAIL: tampered ciphertext → signature invalid
✔ REPLAY: repeated sequence numbers detected
✔ Non-repudiation: offline verification of signed SessionReceipts