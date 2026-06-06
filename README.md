# ExamCode Builder — Online Edition

The full ExamCode Builder, now multi-user and cloud-backed. Host it **free** on GitHub Pages with a **free** Firebase backend. Every builder feature is kept (voice entry, generate table, duplicate detection, files & folders, recycle bin, lookup, settings, overview). Added on top:

- **User login** — each person signs into their own account.
- **Per-user documents** — every user's files/folders are private to them.
- **Super-admin (owner)** — one account that creates and manages all users (passwords via reset link, enable/disable, roles) and sees an **activity log** of what everyone does.
- **No "storage limit" fear** — see the storage strategy below.

---

## Why you will never hit the Firebase storage limit

Your worry was **Firebase Storage** (the blob/file bucket, ~5 GB free). **This app never uses it.** Instead it uses **Cloud Firestore** (the database, **1 GiB free** + 50k reads / 20k writes per day), and it stores data in the most compact way possible:

1. **Text only.** Exam data is just `matricule,examcode` lines — tiny.
2. **Gzip compression.** Before saving, each user's whole workspace is JSON-serialized and **gzip-compressed** (the same `pako` library used in your GCE project). Text like this shrinks ~4–6×.
3. **Document splitting (chunking).** The compressed blob is split into pieces **under Firestore's 1 MiB-per-document limit** and stored as `chunk_0, chunk_1, …`. So even a single giant import (hundreds of thousands of rows) is split safely and **never** overflows a document.
4. **Skip-if-unchanged.** Saves are skipped when nothing changed, so the daily write quota is barely touched.

**Rough scale:** a 1,000-student class ≈ 20 KB raw → ~5 KB compressed. The **1 GiB** free database could hold **tens of thousands** of such files. In practice this is effectively unlimited for exam-code work — and it stays on the free plan.

---

## What you need (all free)

- A Google account (for Firebase).
- A GitHub account (for hosting).
- The two files in this package: **`index.html`** (the app) and **`firestore.rules`** (the security rules).

---

## PART A — Create the Firebase backend (~5 min)

### 1. Create a project
1. Go to <https://console.firebase.google.com> → **Add project**.
2. Name it (e.g. `examcode-builder`), accept defaults, **Create project**. You do **not** need the Blaze plan — the free **Spark** plan is enough.

### 2. Turn on Email/Password login
1. Left menu → **Build → Authentication → Get started**.
2. **Sign-in method** tab → **Email/Password** → **Enable** → **Save**.

### 3. Create the database (Firestore)
1. Left menu → **Build → Firestore Database → Create database**.
2. Choose a location near you → **Next**.
3. Pick **Start in production mode** → **Create**. (We replace the rules next.)

### 4. Paste the security rules
1. Firestore → **Rules** tab.
2. Open **`firestore.rules`** from this package, **copy everything**, and paste it over what's there.
3. **Edit one line:** change `ownerEmail()` to the email you'll use as owner, e.g.
   ```
   function ownerEmail() { return 'patryll@gmail.com'; }
   ```
   (lowercase). **Publish**.

### 5. Get your web config
1. Project **⚙️ (Settings) → Project settings**.
2. Scroll to **Your apps** → click the **web** icon `</>`.
3. Nickname it (e.g. `web`) → **Register app**. **Don't** add Hosting.
4. Copy the `firebaseConfig = { … }` object shown. You'll paste it next.

---

## PART B — Put your keys into the app

Open **`index.html`** in any text editor and scroll to the bottom (the `CLOUD ENGINE` section).

1. **Paste your config** over the placeholder:
   ```js
   const firebaseConfig = {
     apiKey:            "AIza…",            // ← from Firebase
     authDomain:        "your-app.firebaseapp.com",
     projectId:         "your-app",
     storageBucket:     "your-app.appspot.com",
     messagingSenderId: "1234567890",
     appId:             "1:1234:web:abcd…"
   };
   ```
2. **Set the owner email** (must be the **same** email you put in `firestore.rules`):
   ```js
   const OWNER_EMAIL = "patryll@gmail.com";
   ```
3. Save the file.

> The `apiKey` is **safe to publish** — it's not a secret. Your data is protected by the security rules, not by hiding the key.

---

## PART C — Host it free on GitHub Pages (~3 min)

1. Go to <https://github.com/new> → create a repository (e.g. `examcode-builder`), **Public**, **Create**.
2. On the repo page → **Add file → Upload files** → drag in your edited **`index.html`** → **Commit changes**.
   *(You can also upload `firestore.rules` and this `README.md` for safekeeping — they're not served, they're just reference.)*
3. Repo **Settings → Pages**.
4. **Build and deployment → Source: Deploy from a branch.** Branch: **main**, folder: **/ (root)** → **Save**.
5. Wait ~1 minute, then refresh. Your live URL appears, like:
   ```
   https://YOUR-USERNAME.github.io/examcode-builder/
   ```

### Authorize the domain in Firebase (important!)
1. Firebase → **Authentication → Settings → Authorized domains → Add domain**.
2. Add **`YOUR-USERNAME.github.io`** → **Add**.
   *(Without this, login is blocked on the live site.)*

---

## PART D — First run

1. Open your live URL. You'll see the **login screen**.
2. Click **First-time Setup**, enter your **name**, the **owner email** (exactly the one you configured), and a password (6+ chars) → **Create Owner Account**.
3. You're in, as **Super-admin**. An **Admin** tab appears in the menu.
4. Go to **Admin → Create User** to add staff accounts. You set each person's temporary password; share it securely. They can change it later via **Forgot password**.

That's it. Each user signs in and works in their own private workspace; you oversee everyone from **Admin**.

---

## Using the app

- **Builder / Voice / Files / Lookup / Settings / Overview** — exactly as before, but everything now syncs to the cloud automatically (the **“Synced …”** badge shows status). Work follows the user across devices.
- **Admin → Users:** create accounts, **Reset PW** (emails a secure reset link), **Disable/Enable** (instant lockout/restore without losing their work), **Make Admin / Make User**, **Wipe** (deletes a user's data and disables them).
- **Admin → Activity Log:** a running history of logins, saves, imports, exports, deletions and admin actions across all users.

### About passwords (honest note)
From a free, server-less site you can: **set the initial password** when creating a user, and **send a reset link** anytime. Directly *typing a brand-new password into someone else's account* requires Google's Admin SDK (a paid/server feature) and is **not** done here — the reset-link flow is the standard, secure equivalent. **Disable** fully blocks an account immediately.

---

## Security notes

- **Public API key is fine.** Access is enforced by `firestore.rules`, not the key.
- **Rogue sign-ups are harmless.** Because the public API key technically lets anyone create a Firebase auth account, the rules are written so that **data access requires a profile, and only an admin can create profiles.** A stranger who signs up gets an account with **zero** access and is signed straight back out by the app.
- **Owner email must match in both places** (`index.html` *and* `firestore.rules`), or the owner won't get admin rights.
- *(Optional hardening)* In Firebase **Authentication → Settings**, enable **Email enumeration protection**. For extra abuse protection you can later add **App Check**.

## Free-tier limits (you're well within them)

| Resource | Free (Spark) | Typical use here |
|---|---|---|
| Firestore storage | **1 GiB** | KBs per file (gzipped) |
| Reads / day | 50,000 | a handful per login/lookup |
| Writes / day | 20,000 | a few per save (skipped if unchanged) |
| Auth users | unlimited | — |
| Cloud Storage | unused | **0** |

---

## Troubleshooting

- **Owner can't sign in / no Admin tab after setup** → the owner email differs between `index.html` and `firestore.rules`. Make them identical (lowercase), **Publish** rules, reload.
- **“auth/operation-not-allowed”** → you didn't enable **Email/Password** in Authentication (Part A‑2).
- **Login fails only on the live site** → add your `*.github.io` domain to **Authorized domains** (Part C).
- **“Setup needed” banner** → the `firebaseConfig`/`OWNER_EMAIL` placeholders are still in `index.html`.
- **“Could not reach the server”** → no internet, or the device is offline.
- **A user sees no files after a device change** → make sure they're signing into the **same** account; data is per-account.

---

© All rights reserved — **GABSA PATRYLL NYAMAH**
