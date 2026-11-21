# Nexus source curation

Official upstream: <https://github.com/NevermindZZT/letter-shell>

Reviewed upstream base: `03432e824ca889d961743b437bce3f837c47bb86`.
The organization fork retains the upstream implementation and MIT license.

This patch only corrects unsigned key-byte shifts in `src/shell.c`:

- Both `0xFF` key-byte masks use `0xFFU` before left shift.
- Both input-byte shifts convert `data` through `unsigned char` to
  `unsigned int` before shifting.

The initial key-byte offset is 24. Shifting signed `0xFF` by that offset is
undefined even for ordinary ASCII input. A signed `char` with its high bit set
also makes the original input-byte shift undefined. Offsets remain 0, 8, 16 or
24, and the existing short-circuit check still guards the offset-minus-eight
expression. Command matching, parser storage, API and execution policy are
unchanged. No parser is copied into Nexus, and no sanitizer is disabled.

The Nexus `native-asan` profile compiles this file as C11 with
`-fsanitize=address,undefined -fno-omit-frame-pointer`. Reproduction uses the
complete existing `nexus_common_diagnostics_vendor_test` executable:

```sh
ASAN_OPTIONS=detect_leaks=1:halt_on_error=1 \
UBSAN_OPTIONS=print_stacktrace=1:halt_on_error=1 \
/path/to/nexus/build/native-asan/bin/nexus_common_diagnostics_vendor_test
```

`CommonShell.RealParserRunsReadOnlyCommandsAndRejectsMaintenanceNames` exercises
ordinary ASCII commands. Its high-byte input vector also exercises `0x80` and
`0xFF` through the same public task-owned Shell entry. Existing backend reuse,
RTT, frame and resource contracts run in the same complete test executable.
The original source fails UBSan on the first ASCII command; the corrected source
passes without runtime sanitizer diagnostics. Native execution does not qualify
physical terminals, target timing or hardware behavior.
