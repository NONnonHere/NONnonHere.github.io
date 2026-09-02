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
 
**Struct unification**
- [#352](https://github.com/serialport/serialport-rs/pull/352) — Unify `TTYPort` and `COMPort` into a single concrete `SerialPort` struct routed through a `sys` alias.
- [#344](https://github.com/serialport/serialport-rs/pull/344) — Prevent `HANDLE` leaks in `COMPort::open` using `OwnedHandle`.

**Panic-to-Result safety**

- [#322](https://github.com/serialport/serialport-rs/pull/322) — Handle unhandled return values in `TTYPort::open`.
- [#368](https://github.com/serialport/serialport-rs/pull/368) — Return errors instead of panicking when reading baud rate.
- [#359](https://github.com/serialport/serialport-rs/pull/359) — Propagate `tcgetattr` errors from `get_termios_speed` instead of panicking when reading baud rate.
- [#371](https://github.com/serialport/serialport-rs/pull/371) — Fix `TTYPort::into_raw_fd` to transfer the fd safely when unlocking fails.

**Feature enablement**

- [#365](https://github.com/serialport/serialport-rs/pull/365) — Gate platform-specific `Parity` (Mark/Space) and `StopBits` (1.5) variants behind `#[cfg]` and mark the enums `#[non_exhaustive]`.
- [#345](https://github.com/serialport/serialport-rs/pull/345) — Configurable `ReadIntervalTimeout` and prevent timeout overflow on Windows.


Full list: <https://github.com/serialport/serialport-rs/pulls/NONnonHere>
 
---
 
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
