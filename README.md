# 🔏 NotBigBrother

### Age verification that proves you're an adult. Without proving who you are.

---

> Somewhere along the way, "verify your age" became a euphemism for "hand over your government ID, your face, your name, your address, and your browsing habits to a corporation you've never heard of."
>
> We think that's wrong. So we built something different.

---

## The Problem With Every Other Solution

Every major age verification system today is, at its core, a **surveillance pipeline**.

You upload your passport to a third-party service. That service now knows:
- Your real name
- Your date of birth
- Your face (often)
- Which websites asked you to verify your age
- When you visited them
- Probably your IP address and device fingerprint too

That data gets stored. It gets sold. It gets breached. It gets subpoenaed. It gets used to build a profile of you that you never consented to. All just to prove you're over 18.

This is not age verification. **This is mass surveillance with a legal cover story.**

NotBigBrother does it differently.

---

## The Big Idea: Double-Blind Age Verification

We verify your age **once**. You carry the proof. No one connects the two.

- **We** (NotBigBrother) verify your age, but we never know where you're going.
- **The website** gets proof you're an adult, but they never know who you are.
- **Neither party** can correlate your identity with your activity.

That's double-blind. That's the whole thing.

---

## How It Works (Technical Design)

This is built on a **blind signature scheme** — a cryptographic primitive that lets a server sign a message without ever seeing its contents. Here's the full flow:

### Step 1: Age Verification (You and NotBigBrother)

You visit NotBigBrother and prove your age. Currently this is done via face-based age estimation (SCRFD + InsightFace). In production, this would be backed by a stronger identity check (government ID scan, bank verification, etc.) — but the cryptographic flow is identical regardless of the verification method. This is the **only moment your identity is involved**.

At this point:
- We confirm you are 18+
- We issue you a **signed age credential**, a cryptographic token signed by our private key
- We immediately discard any identifying information tied to this session
- We **do not log** your IP address, device fingerprint, or which credential we issued you

The credential itself contains **no personal information**. Not your name, not your birthday. It is simply a message that reads, cryptographically: *"NotBigBrother has verified this person is an adult."*

### Step 2: The Token in Your Hands

You now hold a **one-time-use cryptographic token**. Think of it like an anonymous carnival wristband. It proves you passed the age check at the gate, but it doesn't have your name on it.

Technically, this token is:

```
token = Sign(NBB_private_key, {
  type: "age_verified",
  min_age: 18,
  issued_at: <timestamp>,
  expiry: <timestamp>,
  nonce: <random_unique_value>
})
```

The `nonce` makes every token unique and prevents reuse. The token is stored **locally on your device**, in your browser, in a mobile wallet, wherever. It never touches our servers again.

### Step 3: Presenting the Token (You and the Website/App)

When a website or app needs to verify your age, you present your token. No account. No login. No form fields. You just hand over the token.

The website **cannot** determine:
- Your name
- Your date of birth
- Any other website you've ever verified on
- How you originally got verified
- Any connection to your identity whatsoever

### Step 4: Verification (Website and NotBigBrother's Public Key)

The website validates your token using **NotBigBrother's public key**, which is published openly for anyone to use.

```
valid = Verify(NBB_public_key, token)
```

This is a simple cryptographic check. The website doesn't need to call our servers. They don't need to send us anything. They don't even need to be online to check it. The math works locally.

If the signature checks out, you're verified. If not, rejected. That's it.

NotBigBrother **never learns** that a verification happened, when it happened, or which website requested it.

### The Double-Blind Guarantee

| What NotBigBrother knows | What the Website knows |
|---|---|
| ✅ You passed an age check (once, at issuance) | ✅ This token is cryptographically valid |
| ❌ Where you're going | ❌ Who you are |
| ❌ When you use your token | ❌ How you got verified |
| ❌ How many times you've verified | ❌ Your name, DOB, or any ID data |

Neither party has enough information to build a profile on you. **Neither party can surveil you.** Even if both parties compared notes, there is no linkage to find.

---

## Preventing Double-Use

A valid concern: can someone share their token with others?

Each token includes a unique `nonce` and websites can optionally mark tokens as "consumed" in their own local database after first use. For higher-security contexts, a **nullifier scheme** can be used: compute a one-way hash (e.g. SHA-256) of the token and store that hash. If the same token is presented again, the hash will match and can be rejected. The hash reveals nothing about the underlying token or its owner.

```js
// On the website's side — no calls to NBB required
const nullifier = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(token));
```

This prevents reuse **without** creating a tracking mechanism.

---

## Threat Model: What We Can't Do To You Even If We Wanted To

| Attack | Possible? |
|---|---|
| Sell your identity to data brokers | ❌ We never store it |
| Tell a website who you are | ❌ The token contains no identity |
| Track which sites you verify on | ❌ Verification doesn't involve our servers |
| Build a browsing profile on you | ❌ No mechanism exists to do this |
| Comply with a subpoena for your activity logs | ❌ The logs don't exist |

We are architecturally incapable of surveillance. Not just policy-incapable, *technically* incapable. **You don't have to trust us. You just have to read the code.**

---

## Why Open Source Is Non-Negotiable

Trust-me-bro privacy is not privacy. Every single component of NotBigBrother, the issuance server, the verification library, the client-side wallet, the cryptographic primitives, is fully open source under AGPL-3.0.

This means:
- Security researchers can audit every claim we make
- You can run your own issuance server if you don't trust ours
- Websites can verify our verification library isn't phoning home
- The community can fork, improve, and hold us accountable

If our code doesn't match our promises, you'll know. **That's the point.**

---

## Stack & Cryptographic Primitives

| Component | Technology |
|---|---|
| Signature scheme | RSA blind signatures (Chaum) |
| Age estimation | SCRFD + InsightFace (ONNX, runs on-server) |
| Token format | Custom `nbb1.<base64>` credential file |
| Anonymization primitive | Chaum blind signatures |
| Issuance server | Node.js + Express, open source, self-hostable |

---

## Getting Started

```bash
# Clone the repo
git clone https://github.com/Zonde246/NotBigBrother

# Install dependencies (also downloads ONNX models)
npm install

# Start the server
node server.js
```

Then open `http://localhost:3001` in your browser.

---

## Integrating as a Website or App

Fetch NBB's public key once (cache it — it doesn't change), then verify any credential locally. No API key, no account, no server calls at verification time.

```js
import BlindSignatures from 'blind-signatures';

// 1. Fetch the public key once and cache it
const { N, E } = await fetch('https://notbigbrother-production.up.railway.app/api/issuer/public-key')
  .then(r => r.json());

// 2. Parse the credential file (nbb1.<base64>)
const raw = credentialFileContents.replace(/^nbb1\./, '');
const { token, signature } = JSON.parse(atob(raw));

// 3. Verify the signature locally
const isValid = BlindSignatures.verify({
  unblinded: signature,
  N,
  E,
  message: JSON.stringify(token),
});

if (isValid && token.expiry > Date.now() / 1000) {
  // User is verified 18+. You know nothing else about them.
  // That's the feature.
}
```

---

## FAQ

**Q: What if NotBigBrother gets hacked?**
Our servers hold no identity data post-issuance. A breach reveals nothing about our users.

**Q: Can governments compel you to hand over user data?**
There is no user data to hand over. We comply fully with all legal obligations. They just won't find anything useful here.

**Q: What stops someone from lying about their age to get a token?**
The same thing that stops them with any other verification system: the identity check at issuance. The difference is that only *that check* ever touches your identity. Everything after is anonymous.

**Q: Can I use one token on multiple sites?**
Tokens are designed to be single-use per site, configurable by the integrating service. See the nullifier scheme above.

**Q: Why should I trust you?**
You shouldn't. Not us, not any company, not any government telling you a system is "privacy-respecting" because their marketing team said so. Default trust is how surveillance normalizes itself. Somebody somewhere decided it was fine, and everyone else just went along with it.

You are the final authority on what you trust. That's not a slogan, it's the architecture. The code is open source so you, or anyone you trust to read it, can verify every claim we make. Not because we published a privacy policy. Not because we're a startup with good vibes. Because the math is public, the logic is auditable, and if we ever lie, the proof is right there in the repo for the world to see.

Don't trust us. Read the code. That's the point.

**Q: What happens if I lose my token?**
We cannot recover it. No support ticket, no identity check, no recovery email will get it back. This is not a bug.

Every service that can recover your account can do so because they stored something about you. Recovery is just the customer-friendly name for "we have your data." The ability to recover is proof of surveillance. We have nothing to recover from.

Just verify again. The second check is identical to the first. We still learn nothing about you. You get a new token in two minutes.

Think of it like cash. If you lose a twenty dollar bill the bank cannot give it back, not because they're incompetent, but because cash doesn't have your name on it. That's the feature. Back up your token like you'd back up a password. If you use a password manager, put it there. We will never offer account recovery. The day we do is the day we become everything we're building against.

---

## Contributing

PRs welcome. If you find a flaw in the cryptography, a loophole in the privacy model, or a way this system could be abused, **please open an issue immediately**. Privacy bugs are treated as critical security vulnerabilities.

See [CONTRIBUTING.md] for guidelines.

---

## License

**AGPL-3.0**

We chose the GNU Affero General Public License v3 deliberately and without apology.

MIT and Apache are great licenses for tools you want everyone to freely use in any context. This is not that context. Age verification infrastructure is exactly the kind of code that a bad actor could take, strip the privacy guarantees from, bolt on a data harvesting layer, and deploy as a closed-source surveillance product, all while pointing at our repo to borrow legitimacy.

AGPL-3.0 closes that door. If you use this code, modify it, and run it as a service, **you must publish your modifications under the same license**. No exceptions. No "we just changed a few things" carve-outs. The openness is non-negotiable and it propagates forward.

Fork it. Improve it. Deploy it. Just don't make it someone else's problem.

---

Yes, we used AI to help build this. No, we're not particularly fans of it.

There are inefficiencies in this codebase. There are probably flaws we haven't found yet, and some we don't even have the expertise to recognize. We're not going to pretend otherwise. We are two undergraduate students who got angry enough about something to try to do something about it.

We used AI because we had to. Not as a shortcut, not because it's trendy. This problem is urgent and our bandwidth is finite. Surveillance dressed up as age verification is expanding right now, locking in infrastructure, normalizing data collection on a massive scale. Waiting until we're experienced enough to build this perfectly means waiting until it's too late to matter.

If we can get a secure, working foundation out into the world, something honest, open, and cryptographically sound at its core, and if we can capture the attention of people who actually care about this, then *you* become the project.

You, the developer who's been working in cryptography for a decade. You, the privacy researcher who immediately spotted something we missed. You, the student who knows more about blind signatures than we do and has been looking for something to contribute to. You have knowledge we don't. You have context we don't. This project needs you more than it needs us.

We're not building a finished product. We're building a starting point worth improving.

The codebase is open. The issues tracker is open. The direction is open. If something is wrong, broken, naive, or dangerously half-baked, tell us. PR it. Fix it. This only works if the people who know better show up.

---

*NotBigBrother. Because proving you're an adult shouldn't mean proving everything about yourself.*
