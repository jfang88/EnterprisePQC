For a first pass you want one row per **system**, twelve questions, all dropdowns, five minutes each. Let me build that, plus a script that pre-fills a chunk of it automatically.Now the automation piece — a no-dependency script that pre-fills a chunk of the survey from TLS endpoints.**The quick-scan** is one row per system, ten dropdowns plus a name and a note. Roughly five minutes per system, so 20–30 systems is a half-day of prefill plus a couple of 30-minute workshops.

The simplifications that make it work:

- **One row per system, worst case only.** Not per crypto use. You lose precision and gain completion.
- **Three algorithm buckets instead of an algorithm list.** Does it use public-key crypto (RSA / ECC / both / symmetric-only)? Is anything already broken (3DES, SHA-1, MD5, RC4, TLS < 1.2)? That's enough to rank — you don't need to know it's P-256 versus P-384 to decide whether it goes into pass two.
- **Data lifetime as a band, not a number.** "7–15 years" is answerable in a meeting; "how many years exactly" stalls it.
- **Ease of change is exactly your five options**: config change / library upgrade / code change and release / vendor release required / replace the product or hardware. This is the field that decides sequencing, because the last two need lead time you have to start buying now rather than engineering capacity you can schedule.
- **"Unknown" is a first-class answer** and scores as moderate risk, so gaps surface on the summary instead of quietly scoring as safe.

Output is a triage score, a band 1–4, and a "what kind of move is this" label that combines urgency with difficulty — quick win, long lead time, bundle with refresh, or BAU sweep. Bands 1 and 2 go into the full CBOM survey; the rest wait a cycle.

**The prefill script** answers Q4, Q5 and the evidence note automatically for anything with a TLS endpoint. Feed it `system_id,name,host,port` and it emits CSV with the exact quick-scan headers:


NB:  The scan is optional and may not be run.  It may be faster just to do a survey / workshop for system owners.

```
python3 prefill_quickscan.py targets.csv > prefill.csv
```

It negotiates a default handshake, pulls the certificate key type and size, signature algorithm and expiry, then separately probes whether TLS 1.0/1.1, 3DES or RC4 still work. No third-party packages needed. One thing it gets right that most quick scripts don't: a TLS 1.3 endpoint with an RSA certificate is *both* RSA and ECC, because 1.3 always does (EC)DHE key agreement regardless of the cipher suite name.

**Three other prefill sources worth pulling before the workshops**, roughly in order of effort-to-value:

1. **Certificate Transparency** (`crt.sh`) for your domains gives you the internet-facing certificate population — key type, size, signature algorithm — without touching a single system. It's the fastest way to populate the internet-facing rows.
2. **Your existing SBOM pipeline.** You already have Dependency-Track fed from the Nexus intake, so you can query for the crypto library population — OpenSSL, BouncyCastle, libsodium, wolfSSL, JCE providers — and their versions. That answers Q6 ("where the crypto comes from") for anything built in-house, and the version tells you whether a PQC-capable release even exists. Deriving it from a repository you already control is far cheaper than standing up a crypto discovery tool, and it keeps the CBOM in the same lineage as the SBOM rather than as a parallel artefact.
3. **Cloud and PKI inventories.** KMS/Key Vault key algorithm listings and your AD CS or certificate manager export both dump straight into Q7. These are single CLI calls per account.

What none of that reaches: application-layer crypto, data at rest, batch file transfer, signing keys, mainframe, embedded and OT. Those are the interview questions, and they're the reason the workshop model beats emailing the sheet out.

One caution on reading the results — a clean scan means "not yet detected", never "no crypto". The summary sheet counts low-confidence and unknown rows separately for exactly that reason; report coverage alongside findings or the inventory gets over-trusted the moment it hits a slide.
