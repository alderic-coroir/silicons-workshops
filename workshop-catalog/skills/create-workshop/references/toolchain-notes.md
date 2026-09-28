# Toolchain Notes

**Always read the kernel tree's `debian/control` `Build-Depends` before
assuming a toolchain version.** Different kernel branches have different
requirements. Examples found in practice:

| Kernel | Ubuntu base | clang | rustc | bindgen |
|--------|-------------|-------|-------|---------|
| linux-renesas 7.0 (resolute) | resolute 26.04 | clang-21 | 1.93.1 (native) | bindgen |
| linux-qcom 6.8 (noble) | noble 24.04 | clang-18 | 1.75 (native) | bindgen-0.65 |
| linux-mtk (noble) | noble 24.04 | clang-18 | 1.75 (native) | bindgen-0.65 |
| linux-mtk (jammy) | jammy 22.04 | clang-18 | 1.75 (native) | bindgen-0.65 |

## Jammy (22.04) — clang-18 kernels (e.g. MediaTek 5.15)

Jammy kernels use `clang-18` and `bindgen-0.65`. Use `rustc` (default Jammy),
`rust-src`, `rustfmt`. No LLVM 21 repo needed.

> **Important:** Jammy kernel 5.15 also requires `dwarfdump` (separate package
> from `dwarves`). Without it the build fails:
> `/bin/sh: dwarfdump: not found`

## Noble (24.04) — clang-21 kernels (e.g. Renesas)

These kernels require `clang-21` which is **not in Noble's default archive**.
The `noble-build/hooks/setup-base` adds `apt.llvm.org/noble/` and installs
`clang-21`, `llvm-21-dev`, `rustc-1.91`, `rust-1.91-src`, `rustfmt` (via
`rustfmt-1.91` slave). The `update-alternatives` call registers `rustc-1.91`
as the unversioned `rustc`. **`rust-src` must also be installed** — without it
the kernel's `make rustavailable` fails with "core standard library not found".

## Noble (24.04) — clang-18 kernels (e.g. Qualcomm, MediaTek)

These kernels use `clang-18` (available in Noble's default archive) and
`bindgen-0.65`. Use `rustc` (default Noble 1.75), `rust-src`, `rustfmt`.
No LLVM 21 repo needed. No `update-alternatives` needed.

> **Important:** The `check-health` hook must check for `bindgen-0.65` (not
> `bindgen`). Using the wrong name causes the health check to fail.

## Resolute (26.04) workshops (e.g. Renesas resolute)

Resolute ships `rustc 1.93.1` as the default `rustc` package. No versioned
packages or `update-alternatives` are needed. The `resolute-build/hooks/setup-base`
installs `rustc`, `rust-src`, `rustfmt`, and `bindgen` directly.

## Ubuntu release matching

Always use the workshop variant that matches `ubuntu.release` in config.
If `config/workshop.yaml` says `release: resolute`, use `workshop launch <vendor>-resolute`.
Using the wrong variant (e.g. noble for a resolute kernel) causes toolchain
version failures that are hard to debug.

## Private Launchpad repos and SSH

For `git+ssh://` repos, the workshop container's `~/.ssh/config` must have
`User <launchpad-id>` for `git.launchpad.net`. This is NOT forwarded from the
host's `~/.ssh/config` — only the agent socket is forwarded.
Add a `LAUNCHPAD_USER=<id>` entry to `config/board.env` (gitignored) and ensure
`setup-project` reads it.

## check-config false positives

Some kernels have multi-flavour annotations where one flavour disables Rust.
If `check-config` fails with `CONFIG_RUST_IS_AVAILABLE changed from y to -`
on a non-RT flavour, update the annotation:

```bash
sed -i "s/'arm64-qcom-rt': '-'/'arm64-qcom-rt': 'y'/" \
  kernel-src/debian.qcom/config/annotations
```

## Validation note

`dpkg-deb -c` lists package contents with a `./` prefix (e.g. `./boot/vmlinuz-...`).
The `sdk/deb/lib/validate.sh` boot-image check uses `\.?/boot/` to match both
`/boot/` and `./boot/` correctly.
