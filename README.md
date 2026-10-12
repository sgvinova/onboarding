# onboarding.sgvinova.com: NDA signing site

A static signing page (GitHub Pages) plus a small Cloudflare Worker that records each signature.

```
Client browser ──> onboarding.sgvinova.com   (index.html on GitHub Pages)
        │
        └─ POST signed PDF + details ──> Cloudflare Worker ──> R2 bucket (private vault)
                                                          └──> email to Vinova (Resend)
```

Files in this folder

| File | What it is |
| --- | --- |
| `index.html` | The page: agreement text, signing form, PDF generation |
| `CNAME` | Tells GitHub Pages to serve the site at onboarding.sgvinova.com |
| `worker/worker.js` | Backend that validates, stores and emails each signed copy |
| `worker/wrangler.toml` | Worker settings (bucket, allowed origin, email addresses) |

## 1. Create a separate repository for the page

Each GitHub Pages site can have only one custom domain, and `sgvinova/website` already serves your main site. So make a new repo, for example `sgvinova/onboarding`, and put `index.html` and `CNAME` in its root. Do not put the `worker` folder in a public repo if you later add secrets to it (the secrets below never go in files, but keep the Worker in a private repo as good practice).

Then in the new repo: **Settings > Pages > Source: Deploy from a branch > main / (root)**.

## 2. Point the subdomain at GitHub

In your DNS provider for sgvinova.com, add:

| Type | Name | Value |
| --- | --- | --- |
| CNAME | onboarding | sgvinova.github.io |

Back in **Settings > Pages**, confirm the custom domain shows `onboarding.sgvinova.com`, wait for the DNS check to pass, then tick **Enforce HTTPS**.

## 3. Set up the backend (Cloudflare + Resend)

1. Create a free Cloudflare account and install Wrangler: `npm install -g wrangler`, then `wrangler login`.
2. Create the private bucket: `wrangler r2 bucket create vinova-signed-ndas`
3. Create a Resend account, verify the sgvinova.com domain, and create an API key.
4. In `worker/wrangler.toml`, set `NOTIFY_EMAIL` (where signed copies are sent) and `FROM_EMAIL` (an address on your verified domain).
5. From the `worker` folder, store the key and deploy:
   ```
   wrangler secret put RESEND_API_KEY
   wrangler deploy
   ```
   Wrangler prints the Worker's address, for example `https://vinova-nda-signing.<your-account>.workers.dev`.

## 4. Connect the page to the Worker

In `index.html`, find the `CONFIG` block near the top of the script and set:

- `apiUrl`: the Worker address followed by `/sign`
- `developer`: Vinova's signatory `name` and `title`. Optionally add a transparent PNG of the signature (`signature.png`, next to `index.html`) and set `signatureImage` to `"signature.png"`.

Vinova's signature must be on the page before clients sign, because the Effective Date is the date the last party signs. Commit and push.

## 5. Test before sending to a client

1. Open https://onboarding.sgvinova.com and sign as a test user (use your own name).
2. Confirm the confirmation screen shows a reference and the PDF downloads, and check the PDF reads correctly (signature, name, date, hash line at the end).
3. Confirm the email arrives with the PDF attached.
4. In the Cloudflare dashboard, open the R2 bucket and confirm a folder named with today's date holds the `.pdf` and `.json` for that test.

Test on a phone as well: signing by finger is the usual client experience.

## 6. Editing the agreement later

The agreement text lives in the `AGREEMENT` object in `index.html`. The page, the PDF and the recorded hash all come from it, so edit it there only. If you change the text:

1. Change `AGREEMENT.version` (for example `2026-12-01-v2`).
2. Open the live page, open the browser console, and run `sha256Hex(JSON.stringify(AGREEMENT))` to get the new hash.
3. If you use `EXPECTED_HASH` in `wrangler.toml`, update it and run `wrangler deploy` again. Until you do, signatures against the new text are rejected.

## What is recorded for each signing

Stored in the private R2 bucket: the signed PDF, the signature image, the signer's name, the date they entered, the server timestamp, IP address and country, browser details, the version and SHA-256 hash of the agreement text, and the SHA-256 hash of the PDF. The hashes let you show later that the stored copy is unchanged.

## Things to decide or check

- **Retention and access.** Decide how long you keep records and who can open the bucket. Keep it private, never public. The page tells signers what is stored, but your privacy policy should match.
- **Who the Client is.** The agreement defines the Client as whoever's name is typed in. If clients sign for their companies, a company name field would make the counterparty clearer. Ask your lawyer whether to add one.
- **Abuse protection.** Add a Cloudflare rate-limiting rule on the Worker address so nobody can flood it with fake signatures. Cloudflare Turnstile (a free CAPTCHA) is the next step if you see abuse.
- **No signer email.** Clients download their copy on the confirmation screen and are not emailed one. If you want that, add an optional email field and send a copy with Resend.
- **PDF library.** The page loads jsPDF from cdnjs. If you want the strictest setup, download it into the repo and load it from there.
- **Legal review.** Have a Singapore lawyer confirm that this flow, with its records, is acceptable for your NDAs.
