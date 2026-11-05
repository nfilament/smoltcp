# About this fork

[![Fork CI](https://github.com/nfilament/smoltcp/actions/workflows/fork.yml/badge.svg?branch=buffer-replace-grow)](https://github.com/nfilament/smoltcp/actions/workflows/fork.yml)

This is an unofficial fork of [smoltcp](https://github.com/smoltcp-rs/smoltcp). It is
not affiliated with or endorsed by the smoltcp maintainers or NLnet Labs.

## What it adds

Branch `buffer-replace-grow`, on top of smoltcp `main` at 8b0e5c8 (2026-11-05):

- `tcp::Socket::replace_rx_buffer` and `replace_tx_buffer`: swap in another buffer (for
  example an empty one) once the old one can no longer be needed. The receive buffer
  after the peer's FIN, with everything read; the transmit buffer while it is empty.
  A closing connection can then give its memory back.
- `tcp::Socket::grow_rx_buffer` and `grow_tx_buffer` (and `RingBuffer::grow_into`):
  move a socket's contents into a larger buffer while it is open, out-of-order and
  unacknowledged data included, so connections can start small and grow when
  they carry bulk data (see upstream issue #927).
- A fix for upstream issue #1227, a remotely triggerable panic: a listening socket with
  a receive buffer of 64 KiB or more read its SYN-ACK's unscaled window as scaled, so
  a peer could send up to twice the buffer before the next ACK. The SYN's window is now
  stored in scaled units, and no segment is accepted past the buffer's free space
  whatever the window says.

The commit messages explain the conditions and limits; each change has unit tests.
They come from a userspace TCP/IP stack behind an Android VPN, where thousands of
connections share a phone's memory.

## Why: what it saved

Measured in that stack, which runs smoltcp 0.12 with these changes. Its policy on top
of them:
- each connection's buffers start at 8 KiB and double up to 64 KiB: the receive buffer
  when a read finds it at least half full, the send buffer when a write does not fit;
- a grown send buffer goes back to 8 KiB once it is empty and unused for 30 s (the
  receive buffer keeps its size: shrinking it would take back window already
  advertised);
- both buffers are swapped for empty ones as soon as a connection starts closing.

smoltcp's buffers per connection (a test echoing 2 MiB both ways through the stack):

| | fixed 64 KiB buffers | with this fork |
|---|---|---|
| a new or idle connection | 128 KiB | 16 KiB |
| during a bulk transfer | 128 KiB | 128 KiB |
| idle after a bulk transfer | 128 KiB | 72 KiB |
| closing (FIN received, all read and acknowledged) | 128 KiB until closed | 0 |

The whole stack under load, on Linux, with 1,000 new TCP connections a second through
a local proxy server (peak values):

| | fixed 64 KiB buffers | + buffers given back on close | + buffers start small and grow |
|---|---|---|---|
| heap | 190 MiB | 65 MiB | 42–53 MiB |
| process RSS | 78 MiB | 70 MiB | 53 MiB |

Throughput, latency and connection rates stayed within the runs' spread at each step.
Your numbers depend on how many connections you hold and how they are used; the APIs
only make such a policy possible, they do not choose one.

## CI

`.github/workflows/fork.yml` runs upstream's own `ci.sh` jobs on every push: rustfmt,
checks and tests across the feature sets at the MSRV (1.91) and on stable, clippy, the
unused-code check and the network-simulation tests. A weekly job replays the fork's
commits onto upstream's latest `main` and runs the tests; when it fails, the fork needs
a rebase. Upstream's own workflows are kept unchanged.

## Written with AI, and why it is not upstream

The code, tests and this text were written with an AI assistant (Claude) and
reviewed by the fork's owner. smoltcp follows the
[NLnet Labs LLM policy](https://nlnetlabs.nl/llm-policy/), which does not accept
LLM-generated code or documentation in contributions. We respect that: these changes
are not offered as pull requests, and please do not take them to the smoltcp issue
tracker, forum or chat. Report problems with them here, in this fork.

If smoltcp's maintainers want something like this, they are welcome to design it
their own way; nothing here needs to be reused.

## Using it

```toml
[patch.crates-io]
smoltcp = { git = "https://github.com/nfilament/smoltcp", branch = "buffer-replace-grow" }
```

The fork follows upstream's licence, 0BSD (`LICENSE-0BSD.txt`), and upstream's
copyright notices are unchanged.
