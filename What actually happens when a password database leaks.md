# What actually happens when a password database leaks

**Self-learning article — Cyber Security Essentials**
Topic: password storage, credential attacks, and the defences that survive a breach

---

## Why this topic

Almost every security course starts with "use a strong password," and almost nobody explains what a strong password is actually protecting you *from*. I picked this topic because the answer turned out to be more interesting than the advice. A password's strength barely matters against most attacks. It matters enormously against exactly one: the offline attack that happens after a company's user database has already been stolen.

That gap between the advice and the reason behind it is where most of the real concepts live.

## Hashing is not encryption

The first thing worth getting straight is the difference between two words that people use interchangeably and shouldn't.

Encryption is reversible. There is a key, and whoever holds the key can turn ciphertext back into plaintext. Hashing is one-way by design: a hash function takes input of any size and produces a fixed-length output, and there is no mathematical route back. Feed `hunter2` into SHA-256 and you get 64 hex characters. Feed those 64 characters into anything you like and you will never recover `hunter2`.

This is why a login system should never store your password. It stores the hash. When you log in, the server hashes what you typed and compares the two digests. If an attacker steals the database, they get hashes, not passwords.

Adobe learned the difference the hard way in 2013. Their breach exposed around 153 million records, and the passwords had been *encrypted* rather than hashed — with the same key, in a block cipher mode that produced identical ciphertext for identical passwords. Combined with the plaintext password hints stored alongside, researchers could reconstruct large parts of the password list without breaking the cipher at all. The cryptography was fine. The design decision was not.

## Why a plain hash isn't enough

Here is the part that surprised me most when I started reading about it: for password storage, a fast hash function is a liability.

SHA-256 was built to be fast. That is a virtue when you are verifying file integrity or building a blockchain, and a serious problem when an attacker has your hash file and a GPU. Modern consumer graphics cards compute general-purpose hashes at rates in the billions per second — hashcat benchmarks for a high-end card land somewhere around 10^11 MD5 hashes per second and roughly 10^10 for SHA-256. Those figures shift with every hardware generation, so treat them as orders of magnitude rather than exact numbers, but the order of magnitude is the whole point. An eight-character password drawn from the printable ASCII set has about 6.6 quadrillion possibilities. At 10^11 guesses per second, one card grinds through the entire space in well under a day.

Attackers rarely need the whole space anyway. They start with wordlists — the RockYou dump from 2009 handed the world 32 million real passwords in plaintext, and it is still the standard starting dictionary in CTF challenges and in real attacks. Then they apply rules: capitalise the first letter, append a year, swap `a` for `@`. Most human passwords fall to that within minutes.

Two design fixes address this.

**Salting.** A salt is a random value, unique per user, stored in plaintext next to the hash and mixed into the input before hashing. It does not slow down cracking any single password. What it destroys is the *economy of scale*: without salts, an attacker cracks one hash and instantly identifies every user who chose the same password, and precomputed rainbow tables work against the entire database at once. With unique salts, every account has to be attacked separately. LinkedIn's 2012 breach is the textbook case — unsalted SHA-1, and within days most of the leaked hashes had been reversed.

**Deliberately slow, memory-hard functions.** Instead of a general-purpose hash, password storage should use a key derivation function built to be expensive. bcrypt (Provos and Mazières, 1999) has a tunable cost factor; raise it and each hash takes longer, for defender and attacker alike. scrypt and Argon2 go further by demanding large amounts of memory, which is the resource GPUs and ASICs are worst at scaling. Argon2 won the Password Hashing Competition in 2015, and Argon2id is what OWASP currently recommends for new systems. The defender pays perhaps 100 ms per login. The attacker's billion-guesses-per-second collapses to a few thousand.

| Function | Designed for | Rough GPU throughput | Suitable for passwords |
|---|---|---|---|
| MD5, SHA-1 | speed | ~10^11 /s | no |
| SHA-256 | speed | ~10^10 /s | no, not on its own |
| PBKDF2 | iterated stretching | tunable, GPU-friendly | acceptable, legacy |
| bcrypt | password storage | ~10^5 /s | yes |
| Argon2id | password storage | memory-bound | preferred |

## The attack that doesn't need cracking at all

Everything above concerns offline attacks. There is a cheaper one.

Credential stuffing takes username–password pairs already cracked from one breach and replays them against unrelated services, at scale, automatically. It requires no cryptography. It works purely because people reuse passwords, and its success rate — typically a fraction of a percent per attempt — is more than enough when you have a hundred million pairs to try.

This reframes the usual advice. Password *uniqueness* protects you against credential stuffing; password *complexity* protects you during offline cracking. They are answers to different questions, and uniqueness is the one that matters more in daily life. It is also the one a password manager solves outright, which is why current NIST guidance (SP 800-63B) leans on breach-list screening and length rather than the old forced-rotation and special-character rules that mostly produced `Password1!` and then `Password2!`.

## The layer above the password

If the password is going to be guessed or leaked eventually, the sensible design assumption is that it will be, and something else has to stand behind it.

SMS one-time passwords are the weakest common form of that something, because the second factor lives on a phone number that can be transferred to an attacker through SIM-swap fraud or intercepted through signalling weaknesses in the mobile network. Time-based codes from an authenticator app are meaningfully better — nothing traverses the carrier. But both share a flaw: a convincing phishing page can ask for the code and relay it in real time.

FIDO2 and WebAuthn credentials — passkeys — close that gap by binding the credential to the site's origin in the browser. The private key never leaves the device, and it simply will not sign a challenge for `paytm-secure-login.xyz` when it was registered to `paytm.com`. The user cannot be tricked into approving the wrong site because the check is not left to the user's judgement. That is the property worth internalising: the strongest controls are the ones that don't depend on a human noticing something.

## Why it matters

Password handling is a good lens on security generally because it shows defence in depth working across every layer at once. Hashing assumes the database will leak. Salting assumes the hashes will be attacked. Slow KDFs assume the attacker has better hardware. Multi-factor assumes the password will fall. Origin-bound credentials assume the user will be fooled.

None of these layers is sufficient alone, and each one is designed around the failure of the layer beneath it. That habit — asking "what still holds if this control fails?" — transfers to almost everything else in the field.

## Sources for further reading

- OWASP Password Storage Cheat Sheet — current storage recommendations
- NIST SP 800-63B, *Digital Identity Guidelines: Authentication and Lifecycle Management*
- Provos & Mazières, "A Future-Adaptable Password Scheme," USENIX 1999 (the bcrypt paper)
- Password Hashing Competition (2013–2015) — design rationale for Argon2
- Have I Been Pwned — breach corpus and the k-anonymity range API used for password screening
