+++
title = "GSoC 2026 Final Report (serialport-rs)"
date = 2026-08-29
description = "Replacing serialport-rs's `Box<dyn SerialPort>` API with a concrete struct, and the safety work that rode along with the 5.0 window."

[taxonomies]
tags = ["rust", "gsoc", "serialport-rs"]
+++

<dl class="factsheet">
  <dt>Contributor</dt><dd>Tanmay (<a href="https://github.com/NONnonHere">@NONnonHere</a>)</dd>
  <dt>Organisation</dt><dd>The Rust Foundation</dd>
  <dt>Mentor</dt><dd>Christian Meusel (<a href="https://github.com/sirhcel">@sirhcel</a>)</dd>
  <dt>Project</dt><dd>Improving Ergonomics and Safety of serialport-rs (<a href="https://github.com/serialport/serialport-rs">serialport-rs</a>)</dd>
</dl>

---
 
## Project Goal

serialport-rs returned serial ports as `Box<dyn SerialPort>`, forcing heap allocation and
dynamic dispatch on every user. The project replaces that trait-object design with a single
concrete `SerialPort` struct matching how `std` exposes `File` and `TcpStream` and uses
the resulting major release (5.0) to land safety and ergonomics improvements that were blocked
on a semver-breaking window. All work targets the `playground-5.0` branch.

---
 
## What I Did

## Struct unification

**[#352](https://github.com/serialport/serialport-rs/pull/352) — Unify `TTYPort` and `COMPort` into a concrete `SerialPort` struct**

The library exposed ports through a `SerialPort` trait, so cross-platform code had to hold a `Box<dyn SerialPort>` a heap allocation and dynamic dispatch for what is really just a file descriptor or handle. This PR turns SerialPort into a single concrete struct wrapping the platform type behind a sys alias, and moves platform-specific behaviour into a `SerialPortExt` extension trait. The goal was to make this change without disturbing the common calling pattern: `serialport::new(...).open()` still reads the same, it just no longer needs to box.

**[#344](https://github.com/serialport/serialport-rs/pull/344) — Prevent `HANDLE` leaks in `COMPort::open` using `OwnedHandle`**

Several FFI calls in the Windows open path had their return values ignored, so failures could go unnoticed, and a raw `HANDLE` could leak if `open` returned early during configuration. Rust's `#[must_use]` normally catches this, but FFI bindings rarely have the attribute, so each call had to be checked manually and handled appropriately. Storing the handle as an `OwnedHandle` makes cleanup automatic on any early return.


### Panic-to-Result safety

**[#322](https://github.com/serialport/serialport-rs/pull/322) — Handle unhandled return values in `TTYPort::open`**

The POSIX counterpart to the same problem, return values in the open path that were silently
dropped. Each was analysed and either handled or propagated, so setup failures surface as errors
instead of going unnoticed.

**[#368](https://github.com/serialport/serialport-rs/pull/368) — Return errors instead of panicking when reading baud rate**

`baud_rate()` could panic inside a function that already returns `Result` a `assert!` comparing
input and output speeds across three platform variants, and an `unreachable!()` closing the
PowerPC baud-rate match that was in fact reachable for any port set to a non-listed rate. Both now
propagate as errors rather than terminating the caller's process.

**[#359](https://github.com/serialport/serialport-rs/pull/359) — Propagate `tcgetattr` errors from `get_termios_speed`**

`get_termios_speed` used `.expect()` on a syscall that can fail if the descriptor is invalid or the
device disconnects. What looked like a small panic fix opened a design question on Apple targets
whether to cache the last-set baud rate at all, or read it back live from `termios` including
whether a non-standard rate set via `IOSSIOSPEED` reads back faithfully. Still in review.

**[#371](https://github.com/serialport/serialport-rs/pull/371) — Don't panic in `into_raw_fd` when unlocking fails**

`into_raw_fd` could panic if releasing the advisory lock failed. It now handles that gracefully
while still transferring the descriptor without the destructor closing it.

### Feature enablement

**[#365](https://github.com/serialport/serialport-rs/pull/365) — Platform-specific `Parity` and `StopBits` variants, gated at compile time**

Adds `Parity::Mark`/`Space` and `StopBits::OnePointFive`, but only on the platforms that actually
support them: gated behind `#[cfg]`, with the enums marked `#[non_exhaustive]` so unsupported
variants cannot be constructed where they would fail at runtime. Choosing between compiletime
gating and runtime errors is where most of the time on this went, and it needed evidence rather
than opinion see below.

**[#345](https://github.com/serialport/serialport-rs/pull/345) — Configurable `ReadIntervalTimeout` and prevent timeout overflow**

A Windows fix following a correctness problem I raised while reviewing #320: changing
`ReadIntervalTimeout` without also clearing `ReadTotalTimeoutMultiplier` overflows the total
timeout and breaks the read semantics established in #79.

### Ecosystem analysis

**Downstream usage surveys for the `#[non_exhaustive]` decision**

Marking `Parity` and `StopBits` as `#[non_exhaustive]` is a breaking change for anyone matching on
them exhaustively, so the decision needed data. I surveyed the direct dependents of `serialport`,
then repeated the exercise across the 152 dependents of `tokio-serial`. Setting dominates matching
in both populations 24 setters against 12 apparent parity matchers, and 25 against 7 for
`StopBits`, with only 2 crates reading anything back. Most of the apparent matchers turned out to
be false positives: 13 crates define their own `Parity` enum and match on that to convert into
ours. After checking imports and `From` direction, only 2 crates genuinely match our types. That
result is what settled the design and unblocked the merge.

**Full list of PRs:** <https://github.com/serialport/serialport-rs/pulls?q=is%3Apr+author%3ANONnonHere+>

 
## Current State
 
The concrete `SerialPort` struct is merged on `playground-5.0`; platform-specific behaviour is
exposed through the `SerialPortExt` extension trait. Windows and POSIX ports both use owned
handles (`OwnedHandle` / `OwnedFd`), removing manual cleanup and the associated leak paths.
Several panic sites on the baud-rate and fd-transfer paths now return `Result`.
 
**Merged:** #322, #344, #352, #368
**In review:** #345, #359, #365, #371
 
---
 
## What's Left
 
- Convert `FromRawFd for TTYPort` to a fallible `TryFrom<RawFd>`
- `cargo-minimal-versions` CI job
- Read/write Split
- RS-485 configuration support (deferred to post-5.0)
---

## Reconsidering the Conversion from `FromRawFd`
My proposal listed migrating `FromRawFd` for `TTYPort` to a fallible `TryFrom<RawFd>` 
I don't think that's the right call anymore
- `TryFrom` is safe taking ownership of an arbitrary RawFd without unsafe it reintroduces the double close risk that OwnedFd exists to prevent. This makes the adoption of fallible for locking, but drops the unsafe gate for ownership solving the wrong half.
- `std` precedent the standard library keeps `FromRawFd` and adds safe `From<OwnedFd>` alongside it, rather than merging both into one safe constructor.
- `Flock<OwnedFd>` has no unlocked state so the real open question is whether adopting a foreign descriptor should acquire the lock at all.


## What I Learned
I had little hands-on experience with Win32 and low-level POSIX serial stuff such as file descriptors, handles, ownership and termios / DCB. The process of struct unification taught me those well rather than by pattern matching. The lesson I learned from the project in general was about shipping: I have a tendency to learn something properly before implementing anything and this project helped me overcome that. Beyond the code, I saw the way that a project is managed when it's maintained: CI, reviewing, and clean merging as opposed to just coding. As a mostly lone programmer, the concept of checking in, getting approval, and asking for permission was relatively foreign to me. Building this discipline was one of the most valuable lessons I learned from this project.
 
---
 
## After GSoC
 
I intend to keep contributing to serialport-rs. The first item I plan to pick up is RS-485 support, which I did groundwork on during the program. Also start working on Read/Write Split.
