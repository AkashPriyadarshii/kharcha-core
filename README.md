<!-- Title: kharcha-core — deterministic Rust engine for UPI expense tracking -->
<!-- Description: Rule-based Rust core that turns UPI/SMS/notification text into structured payments. Spam rejection, categorization, exact paise money, triple-signal dedupe, uniffi Kotlin bridge. No AI. -->
<!-- Keywords: upi expense tracker, sms parser rust, upi parser, bank sms parser india, gpay phonepe paytm parser, transaction dedupe, uniffi kotlin, paise money, offline expense tracker, upi number parser -->

<div align="center">
  <img src="assets/icon/app_icon.png" width="96" height="96" alt="kharcha-core logo">
  <h1>kharcha-core</h1>
  <p><strong>Deterministic Rust engine for UPI expense tracking. Text in, payment out.</strong></p>

  <a href="https://github.com/AkashPriyadarshii/kharcha-core/releases"><img src="https://img.shields.io/badge/version-0.1.2-0A6B4D?style=flat-square" alt="Version"></a>
  <a href="https://github.com/AkashPriyadarshii/kharcha-core/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-MIT-0A6B4D?style=flat-square" alt="License"></a>
  <a href="https://github.com/AkashPriyadarshii/kharcha-core/actions"><img src="https://img.shields.io/badge/tests-76%20passing-0A6B4D?style=flat-square" alt="Tests"></a>
  <a href="https://www.rust-lang.org/"><img src="https://img.shields.io/badge/rust-1.96-0A6B4D?style=flat-square" alt="Rust"></a>
  <a href="https://crates.io/crates/kharcha-core"><img src="https://img.shields.io/crates/v/kharcha-core?style=flat-square&color=0A6B4D" alt="crates.io"></a>

  <p>by <a href="https://github.com/AkashPriyadarshii">Akash Priyadarshi</a></p>
  <p>
    <a href="#why">Why</a> ·
    <a href="#quickstart">Quickstart</a> ·
    <a href="#api">API</a> ·
    <a href="#architecture">Architecture</a> ·
    <a href="#non-goals">Non-goals</a> ·
    <a href="docs/INTEGRATION.md">Integrate</a>
  </p>
</div>

[![crates.io](https://img.shields.io/crates/v/kharcha-core?style=flat-square)](https://crates.io/crates/kharcha-core) [![downloads](https://img.shields.io/crates/d/kharcha-core?style=flat-square)](https://crates.io/crates/kharcha-core) [![release](https://img.shields.io/github/v/release/AkashPriyadarshii/kharcha-core?style=flat-square&label=release)](https://github.com/AkashPriyadarshii/kharcha-core/releases)

---

## Why

Every UPI payment in India arrives as text first — an SMS, a push notification, an email receipt. Three parsers (Dart generic, Kotlin generic, a 190-bank fleet) disagreed on the same message, money floated on doubles, and duplicates slipped through reworded carrier footers.

- **One engine.** `engine::parse()` is the only door in. SMS, notification, email body — all one string.
- **Triple-signal dedupe.** UPI ref → content hash (footer-proof, unlike body hashing) → 300s cross-channel window.
- **Exact money.** i64 paise end to end. Both parents float at the edge.
- **Proven, not promised.** 76 tests: 24 unit, 39 parity, 13 notifications. Zero clippy warnings.

## Quickstart

```bash
cargo add kharcha-core
```

```bash
git clone https://github.com/AkashPriyadarshii/kharcha-core
cargo test
```

```bash
cargo run --bin demo -- "INR 340.00 debited from A/C **1234 to ZOMATO UPI:623829102812. Bal: INR 12,400.00"
```

```
ParsedTransaction { payment: ParsedPayment { amount_paise: 34000, merchant: "ZOMATO", is_income: false, upi_ref: Some("623829102812"), balance_paise: Some(1240000), account_mask: Some("1234"), bank_name: None, needs_review: false }, sender: "HDFCBK", timestamp_ms: 0, content_hash: 8485077374821435771 }
```

```bash
cargo clippy --all-targets -- -D warnings
```

| Command | What |
|---|---|
| `cargo test` | Full gate: 76 tests (24 unit + 39 parity + 13 notifications) |
| `cargo run --bin demo -- "<sms>" [sender]` | Parse one message, print the struct |
| `cargo clippy --all-targets -- -D warnings` | Lint gate: zero warnings |
| `cargo build --release` | cdylib (`.so`/`.dll`) for FFI consumers |

## API

| Call | Replaces |
|---|---|
| `parse_capture` / `parse_captures` | Dart `parseUpiNotification`, Kotlin `GenericUpiParser.parse` |
| `check_capture` | `insertCaptured` dedupe (ref → hash → 300s window) |
| `categorize_merchant` + `normalize_merchant_text` | `Categorizer` |
| `apply_filter` | `TransactionFilter.apply` |
| `split_bill`, `parse_amount`, `is_spam`, `encode_inbox_line` | Same-named helpers |

Contracts: money is i64 paise, timestamps are epoch millis, `parse` returns null for spam/casual text by design. Full consumer guide: [docs/INTEGRATION.md](docs/INTEGRATION.md). Agent rules: [docs/AGENT-INTEGRATION.md](docs/AGENT-INTEGRATION.md).

## Architecture

```
src/
  lib.rs             — public API + uniffi scaffolding
  engine.rs          — parse(sms, sender, ts): single entry, batch API, content hash
  parser.rs          — UPI/bank SMS parser (Dart port, fancy-regex lookahead)
  non_transaction.rs — spam rejection (recharge/OTP/loan/request/failure)
  categorize.rs      — merchant normalize + learned-beats-builtin rule match
  dedupe.rs          — ref gate + hash gate + 300s window with ref backfill
  filter.rs          — in-memory transaction filter
  money.rs           — parse_amount → i64 paise
  split.rs           — exact-paise bill split
  ffi.rs             — uniffi surface (Kotlin first); core stays FFI-agnostic
  bin/demo.rs        — smoke CLI
tests/parity.rs      — 35-test Dart-vs-Rust bank SMS corpus (24 unit tests live per-module)
tests/notifications.rs — 40-row UPI-app push notification corpus (GPay/PhonePe/Paytm/Amazon Pay/BHIM/CRED)
bindings/kotlin/     — generated uniffi Kotlin bindings (committed, never hand-edited)
docs/                — INTEGRATION.md (humans), AGENT-INTEGRATION.md (agents)
```

## Non-goals

- **No bank fleet (v1.x).** 190 per-bank parsers stay out until live captures demand them; the generic path covers common formats.
- **No storage, clock, or network.** The core decides; callers own rows, timestamps, and sync.
- **No AI.** Rule maps and regex only. A PR containing "LLM" is rejected.
- **No fraud detection.** Parsing text is this crate's whole job; safety-switch logic lives in the app.
- **No multi-currency amounts yet.** Parser reads ₹/Rs/INR; AED/SGD SMS return null (v1.x).

---

### Ecosystem

- [design-genius](https://github.com/AkashPriyadarshii/design-genius)
- [akash-design-engineering](https://github.com/AkashPriyadarshii/akash-design-engineering)
- [tdlib-android](https://github.com/AkashPriyadarshii/tdlib-android)
- [kharcha](https://github.com/AkashPriyadarshii/kharcha)

### Author

**Akash Priyadarshi** (Patna, Bihar, India) · [GitHub](https://github.com/AkashPriyadarshii) · [Portfolio](https://akashpriyadarshi.vercel.app) · [LinkedIn](https://linkedin.com/in/akash-priyadarshi-1aa51b37a) · [Resume](https://akashpriyadarshii.github.io/Resume/)

### Social

[X / Twitter](https://x.com/Akash__ydv001) · [Threads](https://www.threads.com/@free_dev2026) · [Instagram](https://www.instagram.com/akash.priyadarshii/) · [Reddit](https://reddit.com/user/DragonfruitWeak2801)

*Deterministic UPI parsing in Rust — sms parser, upi expense tracker core, offline-first fintech.*
