# Ember — type a thought, watch it run

A tiny, plain-language programming environment in the spirit of old BASIC, built on
modern, durable foundations. One self-contained HTML file: no install, no account,
no network, no dependencies.

- **[Open Ember](index.html)** — the live environment (language: ember 4.3)
- **[my-page.html](my-page.html)** — the first webpage written *in* Ember
  (by the `favorite-things` program: a list, a loop, and `add … to file`)
- **[The time capsule](ember-capsule_2026-06-11_2.html)** — a sealed copy carrying all six
  frozen language specs, the project charter, the build story, and the 50-program
  compatibility oath; press "Prove the oath" and it replays every sealed program with its
  own engine, byte for byte, from inside the file

## The oath

Ember's founding grievance is that old programs simply stop running. So durability is not a
promise here — it is a tested feature, called **the compatibility oath**: every program ever
shipped with Ember is frozen in a golden corpus with its exact recorded output, and **every
future release must replay all of them byte-for-byte before it may ship.** The corpus only
grows (50 programs as of ember 4.3, reaching back to the first release). It runs against all
three independent engines — JavaScript, PowerShell, and Python — which must agree to the
last byte. A release that breaks one old program does not ship. There are no exceptions, and
there is no expiry.

The full record — six frozen language specs, the project charter, the build story, and the
whole corpus — rides inside [the time capsule](ember-capsule_2026-06-11_2.html). Open it and
press **"Prove the oath"**: the file replays every sealed program with its own engine,
offline, from the inside. It will still be able to do that decades from now.

---

*No relation to the Ember.js web framework — this Ember is a programming language for
people, named for the small spark that starts things.*

Made by Patrick Bradley with Claude.
