# Changelog

Discipline: dated section per tag BEFORE tagging. The release workflow fails
the build when `Cargo.toml` version and tag disagree.

## v0.1.0 — 2026-09-16 (first public release, MIT)

- Unified engine: `parse(sms, sender, ts)` + `parse_batch` + deterministic
  content hash (sender-excluded, footer-proof).
- Full Dart-core port: UPI parser, spam rejection, categorizer, filter,
  money (i64 paise), bill splitter.
- Triple-signal dedupe: ref gate → hash gate → 300s window with ref backfill.
- uniffi Kotlin bridge (`ffi.rs`): 10 fns, 7 records, 1 enum + generated `.kt`.
- Deliberate improvements over Dart: numeric payees kept, voucher promos
  voided, reversals as income, 16 KB input cap, 10k batch/split guards.
- 59 tests green, clippy `-D warnings` clean, two independent audits fixed.
