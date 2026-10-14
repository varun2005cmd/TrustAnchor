# TrustAnchor

TrustAnchor is an educational certificate-authority security pipeline built around
hardware-backed key protection, multi-party authorization, short-lived
certificates, and tamper-evident auditing.

The project models defenses against unauthorized certificate issuance caused by
compromised web servers, gateways, or approval systems.

## Core security controls

| Security concern | TrustAnchor control |
|---|---|
| Private CA keys exposed to application systems | Keys are generated inside PKCS#11 tokens and marked non-extractable |
| A single compromised service can issue certificates | Issuance requires three approvals from a five-person approval group |
| A gateway alters an approved request | The signer independently validates the CSR and re-derives its domains |
| Unauthorized or overly broad domains | Signer-side allowlists, wildcard rejection, and validity limits |
| Missing or modified security records | Hash-chained audit logging, including rejected requests |
| Long certificate lifetimes | Certificates are limited to 24–72 hours and include OCSP Must-Staple |

## Project status

- [x] HSM-backed root and issuing CA using SoftHSM2 and PKCS#11
- [x] Multi-party approval gateway and signer
- [x] Automated attack and integrity tests
- [ ] Shamir-secret sharing for the HSM PIN
- [ ] Sealed signer service
- [ ] Network separation and mutual TLS
- [ ] ACME-based renewal automation
- [ ] OCSP responder and emergency revocation
- [ ] Hard-fail client validation
- [ ] Issuance monitoring
- [ ] End-to-end attack simulation

## Architecture

```text
requester.py --> gateway.py --> signer.py --> HSM
                  approval       policy       issuing key
                  collection     checks
approver.py ----> signatures
```

The gateway coordinates requests and approvals but is not trusted to enforce
certificate policy. The signer independently verifies the request, validates
each approval, checks the CSR, applies issuance restrictions, and requests the
signature from the HSM.

## Repository structure

```text
ca_sign.py        Creates root and intermediate CA certificates
sign_leaf.py      Performs a standalone leaf-certificate issuance test
extract_test.py   Verifies that the issuing key cannot be extracted
env.sh            Defines the PKCS#11 library location
certs/            Stores public certificates, public keys, and CSRs

mofn/
  common.py       Canonical request encoding and signature verification
  keygen.py       Creates the five approval keys and public-key registry
  signer.py       Applies policy, verifies approvals, signs, and audits
  gateway.py      HTTP service for collecting approvals
  requester.py    CLI for submitting and issuing certificate requests
  approver.py     CLI for reviewing and approving requests
  attack_tests.py Automated security and tamper-resistance tests
```

Private keys are excluded from version control. All PINs used in this project are
for local laboratory demonstrations only.

## Requirements

- Ubuntu on WSL2 or native Linux
- Python 3.10 or later
- OpenSSL 3.x
- SoftHSM2
- OpenSC and `pkcs11-tool`

Install the required packages:

```bash
sudo apt update
sudo apt install -y softhsm2 opensc openssl python3-venv python3-full \
                    python-is-python3 build-essential python3-dev libssl-dev git
```

## Installation

Keep the project inside the Linux filesystem, such as `~/trust-anchor`, rather
than under `/mnt/c`.

### 1. Create the Python environment

```bash
cd ~/trust-anchor
python3 -m venv venv
source venv/bin/activate
pip install python-pkcs11 cryptography
```

### 2. Configure SoftHSM2

Two independent tokens are used: one for the root CA and another for the
issuing CA.

```bash
mkdir -p ~/trust-anchor/root-hsm/tokens \
         ~/trust-anchor/issuing-hsm/tokens \
         ~/trust-anchor/certs

cat > ~/trust-anchor/root.conf <<EOT
directories.tokendir = $HOME/trust-anchor/root-hsm/tokens
objectstore.backend = file
log.level = ERROR
EOT

cat > ~/trust-anchor/issuing.conf <<EOT
directories.tokendir = $HOME/trust-anchor/issuing-hsm/tokens
objectstore.backend = file
log.level = ERROR
EOT

cat > ~/trust-anchor/env.sh <<'EOT'
export P11MOD=/usr/lib/softhsm/libsofthsm2.so
EOT

source ~/trust-anchor/env.sh
```

If the PKCS#11 library is installed elsewhere, locate it with:

```bash
ls /usr/lib/*/softhsm/
```

Update `env.sh` with the correct path if necessary.

### 3. Generate the CA keys

```bash
export SOFTHSM2_CONF=~/trust-anchor/root.conf
softhsm2-util --init-token --free --label rootCA \
  --so-pin 11111111 --pin 1234

pkcs11-tool --module "$P11MOD" --token-label rootCA --login --pin 1234 \
  --keypairgen --key-type EC:secp384r1 \
  --label root-key --id 01

export SOFTHSM2_CONF=~/trust-anchor/issuing.conf
softhsm2-util --init-token --free --label issuingCA \
  --so-pin 22222222 --pin 5678

pkcs11-tool --module "$P11MOD" --token-label issuingCA --login --pin 5678 \
  --keypairgen --key-type EC:secp384r1 \
  --label issuing-key --id 02
```

### 4. Export public keys

Only public key material is exported from the tokens.

```bash
export SOFTHSM2_CONF=~/trust-anchor/root.conf
pkcs11-tool --module "$P11MOD" --token-label rootCA --login --pin 1234 \
  --read-object --type pubkey --label root-key \
  -o certs/root_pub.der

openssl pkey -pubin -inform DER \
  -in certs/root_pub.der \
  -out certs/root_pub.pem

export SOFTHSM2_CONF=~/trust-anchor/issuing.conf
pkcs11-tool --module "$P11MOD" --token-label issuingCA --login --pin 5678 \
  --read-object --type pubkey --label issuing-key \
  -o certs/issuing_pub.der

openssl pkey -pubin -inform DER \
  -in certs/issuing_pub.der \
  -out certs/issuing_pub.pem
```

### 5. Create the CA certificates

The CA private keys remain inside the tokens while the HSM performs the
cryptographic signing operation.

```bash
export SOFTHSM2_CONF=~/trust-anchor/root.conf
python ca_sign.py root
python ca_sign.py intermediate

openssl verify -CAfile certs/root.crt certs/root.crt
openssl verify -CAfile certs/root.crt certs/issuing.crt
```

Both verification commands should return `OK`.

After creating the issuing certificate, the root token can be removed from the
active environment, encrypted, and stored offline.

### 6. Generate approval keys

The demonstration uses a three-of-five approval threshold.

```bash
cd ~/trust-anchor/mofn
mkdir -p keys
chmod 700 keys
python keygen.py
```

### 7. Generate a test CSR

```bash
cd ~/trust-anchor

openssl req -new \
  -newkey ec \
  -pkeyopt ec_paramgen_curve:prime256v1 \
  -nodes \
  -keyout certs/leaf.key \
  -subj "/CN=test.local" \
  -addext "subjectAltName=DNS:test.local" \
  -out certs/leaf.csr
```

## Running the demonstration

Open two terminals and prepare both environments:

```bash
cd ~/trust-anchor
source env.sh
source venv/bin/activate
export SOFTHSM2_CONF=~/trust-anchor/issuing.conf
export HSM_PIN=5678
cd mofn
```

### Terminal 1: start the gateway

```bash
python gateway.py
```

### Terminal 2: submit a request

```bash
python requester.py submit ../certs/leaf.csr 24 web1
```

Save the returned request ID:

```bash
RID=<request-id>
```

An issuance attempt without sufficient approval must be rejected:

```bash
python requester.py issue "$RID" ../certs/test.crt
```

Approve the request with three different approvers:

```bash
python approver.py approver1 "$RID"
python approver.py approver2 "$RID"
python approver.py approver3 "$RID"
```

Each approver is shown the domains extracted from the CSR and must confirm the
first domain.

Issue and verify the certificate:

```bash
python requester.py issue "$RID" ../certs/test.crt

openssl verify \
  -CAfile ../certs/root.crt \
  -untrusted ../certs/issuing.crt \
  ../certs/test.crt

openssl x509 -in ../certs/test.crt -noout -text |
  grep -E "Not After|status_request|DNS:"
```

A second attempt to issue the same request must be rejected as a replay.

## Expected behavior

| Test | Expected result |
|---|---|
| Zero approvals | Request rejected |
| Two approvals | Request rejected |
| Three distinct valid approvals | Certificate issued |
| Certificate validity | Approximately 24 hours |
| Certificate extensions | OCSP Must-Staple and `DNS:test.local` |
| Repeated issuance | Rejected as a replay |

## Security and integrity tests

### Verify key protection

```bash
export SOFTHSM2_CONF=~/trust-anchor/issuing.conf

pkcs11-tool --module "$P11MOD" \
  --token-label issuingCA \
  --login \
  --pin 5678 \
  --list-objects

python ../extract_test.py
```

The output should indicate that the key is non-extractable and that reading the
private key value is refused.

### Run the attack suite

```bash
cd ~/trust-anchor/mofn
python attack_tests.py
```

The suite covers:

1. Successful issuance with three approvals
2. Approval replay
3. Insufficient approvals
4. Duplicate approval attempts
5. Forged signatures
6. CSR replacement after approval
7. Request modification after approval
8. Expired requests
9. Excessive certificate validity
10. Domains outside the configured policy

The expected result is:

```text
10/10 tests behaved as expected
audit chain intact: True
```

### Test audit-log tampering

Modify one character in `mofn/audit.log`, then run:

```bash
python -c "import signer; print(signer.verify_audit())"
```

The verification routine should identify the broken chain entry.

## Troubleshooting

| Problem | Resolution |
|---|---|
| Requester reports connection refused | Start `gateway.py` first |
| `KeyError: P11MOD` | Run `source ~/trust-anchor/env.sh` |
| `KeyError: HSM_PIN` | Export `HSM_PIN=5678` |
| `NoSuchToken` | Check `SOFTHSM2_CONF` |
| `ModuleNotFoundError: common` | Run commands from `mofn` |
| Requests remain marked as replayed | Remove `mofn/signer_state.db` for a fresh demonstration |
| PKCS#11 provider causes an OpenSSL crash | Use the Python PKCS#11 implementation |
| OpenSSL reports explicit ECC parameters | Use the Python PKCS#11 signing path |

## Known limitations

- SoftHSM2 provides software emulation rather than physical tamper resistance.
- Production deployments should use a hardware or cloud HSM.
- Approval keys are stored as software keys in this demonstration.
- Production approvers should use hardware tokens or WebAuthn.
- The gateway and signer currently share one host.
- A compromised signer host may still attempt to access the HSM.
- Browser enforcement of OCSP Must-Staple is inconsistent.
- Short certificate lifetimes are the primary protection against delayed
  revocation.

## Disclaimer

TrustAnchor is an educational prototype and is not intended for production
certificate-authority operations without additional security review, isolation,
monitoring, key management, and operational controls.