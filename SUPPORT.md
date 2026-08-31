# Support

Start here when something does not build, program, or behave as expected:

```text
agamemnon --version
agamemnon doctor --no-hardware
```

For the current feature list and known failures, see
[docs/STATUS.md](docs/STATUS.md).

When hardware is connected, run `agamemnon doctor`. UART probing resets the
chip into the mask ROM, so it only runs when you explicitly provide
`--uart-port PORT`.

## Reference hardware

Most hardware testing is done on:

- `AG32VF303CCT6`, LQFP-48;
- fabric target `AGRV2KL48`;
- board definition `agamemnon/sdk/boards/ag32vf303-l48.toml`;
- AGM-compatible CMSIS-DAP with AGaMEMnon's OpenOCD build.

Other AG32 packages are useful research targets, but they have much less board
testing.

## Where to look

| Problem | Read first |
|---|---|
| What is this chip? | [AG32 overview](docs/AG32_OVERVIEW.md) |
| Install/tool not found | [Installation](docs/INSTALLATION.md) |
| Build/command syntax | [Usage](docs/USAGE.md) |
| Project manifest | [Projects](docs/PROJECTS.md) |
| DAP, USB, UART, flash, recovery | [Programming](docs/PROGRAMMING.md) |
| Unsupported route/primitive | [Status](docs/STATUS.md) |
| Board wiring / measured hardware behavior | [Hardware validation](docs/HARDWARE_VALIDATION.md) |
| Known-good board/probe setup | [Known-good hardware](docs/KNOWN_GOOD_HARDWARE.md) |
| Provenance / redistribution | [Notices](NOTICE.md) |

## Before filing an issue

Please collect:

```sh
agamemnon --version
agamemnon doctor --json --no-hardware
git rev-parse HEAD
git status --short
```

For hardware problems, also include:

- exact chip/package and board revision if known;
- operating system;
- probe/transport and relevant wiring;
- full command, with secrets and personal paths removed;
- complete error output;
- what you expected and what actually happened;
- whether you changed flash;
- whether DAP or mask-ROM UART still responds.

For fabric failures, the routed JSON, confidence/pack report and image hash are
usually much more useful than a screenshot of the terminal.

Do not attach factory flash dumps publicly without checking them for unique data
and redistribution problems.

## Programming

For a new fabric design, prefer SRAM first. It is faster to iterate and leaves
flash alone.

Before persistent writes:

1. read [Programming](docs/PROGRAMMING.md);
2. make a full flash backup;
3. preserve the decompressor/configuration layout;
4. verify changed bytes by readback;
5. make sure you still have a recovery path.

The flash-resident USB uploader is convenient but stops being useful if main
flash is broken. SWD/DAP and the mask-ROM UART are the recovery-oriented paths.

## Asking for help

Use GitHub issues for reproducible bugs, documentation problems and hardware
results. Security issues go through [SECURITY.md](SECURITY.md), not the public
issue tracker.