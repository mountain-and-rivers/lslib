# Linux fix for `Divine`'s path validation

This fork applies one fix on top of [Norbyte/lslib](https://github.com/Norbyte/lslib):
`Divine`'s CLI crashed with an unhandled `System.InvalidOperationException` on
Linux for **every** action that takes `-s`/`-d` (extract, convert-resource,
convert-loca, etc).

## Root cause

`CommandLineActions.TryToValidatePath()` validated that a path was absolute by
calling `Uri.TryCreate(path, UriKind.RelativeOrAbsolute, out uri)` and then
checking `uri.IsFile`. On Linux, a perfectly valid absolute path like
`/tmp/foo` parses as a *relative* URI (`IsAbsoluteUri == false`), and reading
`.IsFile` on a relative `Uri` throws `InvalidOperationException`. This
reproduces for every genuinely absolute Unix-style path - it isn't about
which arguments are passed, just about running on Linux at all. It works
fine on Windows, where `C:\...` parses as an absolute file URI, which is why
this had gone unnoticed upstream.

## The fix

`Path.IsPathRooted(path)` is the real cross-platform "is this absolute path"
check the original code was going for; the extra `Uri`/`IsFile` check was an
unstated Windows-only assumption. Replaced the whole `Uri`-based check with
a plain `Path.IsPathRooted(path)` check in
[`Divine/CLI/CommandLineActions.cs`](Divine/CLI/CommandLineActions.cs).

## Status

Found and fixed while modding Baldur's Gate 3
([mountain-and-rivers/bg3-class-mod](https://github.com/mountain-and-rivers/bg3-class-mod))
from a Linux-based cloud dev session, where hand-editing a binary `.lsf`
resource needed a working `Divine` CLI. Not yet built/tested end-to-end from
this checkout on Linux - the full `LSTools.sln` build needs `GpLex`/`Gppg`
(Windows binaries under `external/gppg/binaries/`) to regenerate the story
goal/header parsers, which aren't available in that cloud session. The fix
itself was verified by decompiling the vendored `Divine.dll` from
`bg3-class-mod`, applying the same one-line change, rebuilding just the
`Divine` project against the existing prebuilt `LSLib.dll`, and confirming it
runs `convert-resource` successfully on Linux (see
`lslib/Tools/DivineLinux/README.md` in that repo for the exact steps and a
working prebuilt copy). This fork carries the equivalent fix at the actual
source location for anyone building from source, and as a base for a PR back
upstream.
