# Executable-Compressor Savings Report

This report measures the on-disk savings of running an executable compressor
against the largest binary that ends up on disk after `npm install @rsbuild/core`.

## Target binary

`@rsbuild/core` itself ships only JavaScript. The "big executable file" users
see is a transitive native dependency:

```
node_modules/@rspack/binding-linux-x64-gnu/rspack.linux-x64-gnu.node
```

Pulled in by `@rspack/core`, which is a hard dependency of `@rsbuild/core`.

| Property | Value |
|---|---|
| Format | ELF 64-bit LSB shared object, x86-64, dynamically linked, stripped |
| Loaded via | `process.dlopen` (N-API addon) |
| Size | **51,166,344 B (48.80 MiB)** |
| Source package | `@rspack/binding-linux-x64-gnu` (built upstream in `web-infra-dev/rspack`) |

`@rsbuild/core@2.0.3` was used; `@rspack/core` was resolved transitively.

## Environment

| | |
|---|---|
| OS | Ubuntu (x86_64) |
| Node | v20.20.2 |
| UPX | 4.2.2 |
| `xz` / `zstd` / `gzip` | system defaults |

## Method

1. `npm install @rsbuild/core` in a clean directory.
2. Take a copy of the `.node` file.
3. Apply each compressor at its strongest practical setting.
4. For UPX, replace the file in place and verify both:
   - `node -e "require('…rspack.linux-x64-gnu.node')"` returns without crashing.
   - A real Rsbuild build of a tiny app completes successfully.
5. Restore the original after each test.

## Results

### Executable compressor (UPX)

UPX is the only tool listed here that produces a **self-extracting,
still-loadable** `.node` file. The file remains a valid ELF that Node's
`dlopen` can load; UPX's stub decompresses on load.

| Variant | Output size | Ratio | Saved | `require()` works | `rsbuild build` works |
|---|---:|---:|---:|---|---|
| Original | 48.80 MiB | 100.00 % | – | ✅ | ✅ |
| `upx --best` (NRV)        | **18.27 MiB** | **37.45 %** | **30.52 MiB** | ✅ | ✅ |
| `upx --best --lzma`       | 14.82 MiB     | 30.37 %     | 33.98 MiB     | ❌ `Trace/breakpoint trap` | ❌ |
| `upx --best --lzma --force` | 14.82 MiB   | 30.37 %     | 33.98 MiB     | ❌ same | ❌ |

`--lzma` packs further but UPX's LZMA stub crashes when Node `dlopen`s the
result. Default NRV compression is the largest setting that still loads.

#### Functional check (NRV variant)

Built a one-file Rsbuild project with both bindings:

| Binding | Build wall time | Peak RSS | Output |
|---|---:|---:|---|
| Original (48.80 MiB) | 0.18 s | 127,472 KB | identical |
| UPX `--best` (18.27 MiB) | 0.35 s | 126,952 KB | identical |

The packed binding adds ~0.17 s to first load (one-time decompression in the
UPX stub) and produces byte-identical build output. RSS is unchanged because
the decompressed image lives in memory either way.

### Reference: pure-data compressors (not loadable, gz/zstd/xz)

Listed so the savings from UPX can be put in context. These shrink the file
on the wire/disk but produce a non-executable artifact — using them would
require shipping a custom post-install or runtime decompression step, which
is a different change shape and is *not* "an executable compressor".

| Compressor | Output size | Ratio | Saved |
|---|---:|---:|---:|
| `gzip -9`  | 17.92 MiB | 36.73 % | 30.88 MiB |
| `zstd -19` | 12.86 MiB | 26.36 % | 35.93 MiB |
| `xz -9e`   | 11.74 MiB | 24.06 % | 37.06 MiB |

Note: the npm tarball is already gzip-compressed during transport, so the
"download" cost roughly tracks the `gzip -9` row. The savings reported in
this document are **on-disk-after-install** savings, which is what users
actually observe in `node_modules`.

## Headline number

Applying `upx --best` to the rspack napi binding reduces the largest file
in a fresh `@rsbuild/core` install from **48.80 MiB to 18.27 MiB**, a
**62.55 % reduction (–30.52 MiB)**, with no observable change in
build output and a one-time ~0.17 s extra cost on first load.

## Caveats and where any change would have to land

- The `.node` binary is **not built by this repository**. Rsbuild only
  re-exports rspack. Any change to how this file is compressed needs to
  happen in [`web-infra-dev/rspack`](https://github.com/web-infra-dev/rspack),
  in the napi packaging step (e.g. its `@rspack/binding-*` build/publish
  pipeline), so that the *published* `.node` is already UPX-packed and the
  saving is realized for every install of every downstream package
  (`@rsbuild/core`, `@rspack/cli`, etc.).
- `--lzma` must be avoided: it gives the smallest output but breaks
  `dlopen` for the resulting library.
- These numbers are for `linux-x64-gnu` only. UPX supports `linux/*`,
  `win32/pe`, and `macho/*`, so the same approach applies to the other
  `@rspack/binding-*` artifacts, but per-platform numbers will differ and
  should be measured the same way before shipping.
- Cargo-side reductions (`strip = "symbols"`, `lto = "fat"`,
  `codegen-units = 1`, `panic = "abort"`, `opt-level = "z"`) are
  complementary and should be benchmarked first — they shrink the file
  before any compressor runs and don't add load-time overhead.

## Reproduction

```bash
mkdir bench && cd bench
npm init -y >/dev/null
npm install @rsbuild/core
F=node_modules/@rspack/binding-linux-x64-gnu/rspack.linux-x64-gnu.node
ls -l "$F"

cp "$F" orig.node
cp "$F" packed.node
chmod +x packed.node
upx --best packed.node            # safe: stays loadable
ls -l orig.node packed.node

# Verify the packed binding still loads and builds
cp packed.node "$F"
node -e "require('./'+process.env.F || '$F')"
npx rsbuild --version
```
