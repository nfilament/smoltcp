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

The commit messages explain the conditions and limits; each change has unit tests.
They come from a userspace TCP/IP stack behind an Android VPN, where thousands of
connections share a phone's memory.

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
