# Platform support

Linux and macOS builds are produced for amd64 and arm64. Linux retains its
`/proc` backend; macOS uses the system `lsof` and `ps` tools without adding Go
dependencies or requiring CGO. Linux releases include both DEB and RPM packages.

## macOS behavior

- TCP/UDP, IPv4/IPv6, process ownership, CPU sampling and process details are supported.
- CPU usage is derived from successive cumulative `ps` CPU-time samples. Each PID
  is sampled once per collection, even when it owns multiple sockets.
- System commands have a five-second timeout and use the C locale for parsing.
- Run with `sudo` to include other users' processes. Non-root scans can omit sockets
  entirely; they are not necessarily returned as rows with an unknown PID.
- Systemd units and Docker Desktop VM process metadata are unavailable.
- Thread count is unavailable and displayed as `-`. Other process details are
  best-effort when a process exits or access is restricted.
- Scans and CPU sampling can be slower than Linux because they invoke system tools.

## Validation performed locally

- Linux unit tests and end-to-end tests passed.
- `go vet` and `gofmt` passed in a temporary checkout with LF line endings.
- macOS binaries and all unit-test binaries compile for Intel and Apple Silicon.
  Cross-compilation does **not** run the macOS tests.
- `goreleaser check` and the snapshot release passed, producing four binary
  archives, two DEB packages, two RPM packages and checksums.
- Installer simulations verified archive selection, checksum validation and
  extraction for Linux/macOS on both architectures.
- Native macOS socket/process tests and end-to-end tests are configured on
  `macos-15` and `macos-15-intel` in GitHub Actions. They have not been run here.
- `go test -race -p 1 -count=1 ./...` passed after installing GCC 13.3 and
  development headers under `~/.local/share/portop-toolchain`. Packages were
  run sequentially because the baseline test observes all system listeners;
  concurrent socket tests can change its snapshot. No Go data race was reported.

Existing CRLF line endings in the working tree were preserved. Verification used
a temporary copy normalized to LF, matching the repository's committed files.
No release was published.

## Implementation references

- [lsof machine-readable output](https://github.com/lsof-org/lsof/blob/master/Lsof.8)
- [Apple ps manual](https://github.com/apple-oss-distributions/adv_cmds/blob/main/ps/ps.1)
- [GoReleaser nFPM packaging](https://www.goreleaser.com/customization/package/nfpm/)
- [GitHub macOS runner architectures](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)

## Local C compiler

The user-local compiler does not require changes to system packages. To run the
race checks with it:

```sh
CGO_ENABLED=1 CC="$HOME/.local/share/portop-toolchain/bin/gcc" \
  go test -race -p 1 -count=1 ./...
```

Run these checks separately from other suites that open listening sockets.
