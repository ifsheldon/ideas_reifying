# WASIp2 tutorial update checklist

Reviewed: 2026-09-22.

Apply each article change to both [English](content/blog/complete-guide-to-wasip2-for-rust-python-programmers/index.en.md) and [Chinese](content/blog/complete-guide-to-wasip2-for-rust-python-programmers/index.md).
This records the source audit against the working `wasi_mindmap` examples.
Items 1–4 are complete in both languages; the basic, dynamic, and KV host snippets were extracted from each article, compiled, and run independently with fresh guest artifacts before the final editorial trim.
The final articles omit detailed setup instructions and repeated `mod utils;` declarations in the dynamic and KV examples, reusing the initial Rust host context.
The remaining items are still pending.
Line numbers below refer to the article before these fixes.

## Order and scope

Prioritize direct code replacements, then small corrections, then changes requiring workflow explanations or image editing.
Size estimates include updating both languages: **XS** means a few lines, **S** means one snippet or several local edits, and **M** means a section or multiple related snippets.
The order reflects implementation effort, not severity.

| Rank | Fix | Size | Difficulty | Main work |
| --- | --- | --- | --- | --- |
| 1 | Dependency versions and error context | XS | Low | Replace version strings and an import |
| 2 | Basic Rust host entrypoint | S | Low | Add missing module declaration and engine initialization |
| 3 | Dynamic function lookup and post-return | S | Low | Swap in the working lookup/call code |
| 4 | KV host APIs | S | Low | Replace obsolete macro and linker syntax |
| 5 | KV guest connection lifetime | S | Low–medium | Copy the persistent-connection implementation |
| 6 | Small command/link corrections and metadata | XS | Low | Correct a command, link, and update notes |
| 7 | Command-component workflow | M | Medium | Replace build setup and code, then adjust explanations |
| 8 | Python setup, paths, and loader explanation | M | Medium | Make commands executable in a clearly defined directory layout |
| 9 | Command-component diagram label | S–M | Medium | Correct text embedded in an image |

Ranks 1–4 were completed together as one coherent Wasmtime 49 update, including removal of the old manual `post_return` call.
Keep the unresolved-issue list unchanged for this update.

## 1. Dependency versions and error context

- [x] Use Rust `wasmtime = "49.0"`, `wasmtime-wasi = "49.0"`, and `wit-bindgen = "0.62.0"`, matching the tested repository.
- [x] Replace `use anyhow::Context;` with `use wasmtime::error::Context;` and remove the standalone `anyhow` dependency if no article code still uses it.
- [x] Keep Python Wasmtime at the repository's separate pin, `38.0.0`; do not change it to match the Rust crate version.

Locations: EN 177, 287–306; ZH 173, 280–299.
Reference: `wasi_mindmap/Cargo.toml`, `host-rs/Cargo.toml`, and `host-rs/src/utils.rs`.

## 2. Basic Rust host entrypoint

- [x] Add `mod utils;` to the shown `src/main.rs`.
- [x] Initialize `let engine = Engine::default();` and pass `&engine` to the helper.
- [x] Align the WIT and component artifact paths with the repository's shared `wit-files` and `target` layout.
- [x] Register WASI imports for the documented debug guest build, using the shared helper and a mutable linker.

Locations: EN 371–394; ZH 364–387.
Reference: `wasi_mindmap/host-rs/src/main.rs` and `host-rs/src/adder/rs_guest/sync_version.rs`.
Validation: compile and run the host snippet using the repository layout.

## 3. Dynamic function lookup and post-return

- [x] Destructure `get_export` results as `(_, interface_idx)` and `(_, func_idx)`; Wasmtime 49 returns an item/index pair.
- [x] Remove `typed_func.post_return(...)` and its “required” comment; cleanup is automatic with the updated Wasmtime version.
- [x] Synchronize result-type inspection between the languages.
- [x] Use the public paths `wasmtime::component::ComponentExportIndex` and `wasmtime::component::Func` in the explanation.

Locations: EN 619–665; ZH 611–661.
Reference: `wasi_mindmap/host-rs/src/adder/rs_guest/interfaced_sync_version.rs`, function `run_adder_dynamic`.
Validation: run the dynamic adder example and check the result is 3.

## 4. KV host APIs

- [x] Replace `trappable_imports: true` with `imports: { default: trappable }`.
- [x] Import `HasSelf` and use `KvDatabase::add_to_linker::<_, HasSelf<_>>(&mut linker, |s| s)?;`.
- [x] Call the repository helper with its missing argument: `bind_interfaces_needed_by_guest_rust_std(&mut linker, false);`.
- [x] Show the helper and `mod utils;` declaration in the initial Rust host example for reuse by the KV host.

Locations: EN 769–860; ZH 764–854.
Reference: `wasi_mindmap/host-rs/src/kv_store/rs_guest/sync_version.rs` and `host-rs/src/utils.rs`.
Validation: compile and run the updated KV host against the guest.

Validation of items 1–4 on 2026-09-22, before the editorial trim: all six extracted host snippets passed with `wasmtime 49.0.0`, `wasmtime-wasi 49.0.0`, and `wit-bindgen 0.62.0` using the repository lockfile and offline dependency cache.
The basic and dynamic hosts asserted `3`; dynamic lookup also printed both parameter types and the result type.
The KV hosts asserted `None` for one call; persistence across calls remains item 5.
Additional debug builds passed for the interfaced adder and the article's own KV guest, using the repository layout and explicit manifest paths.
`zola build` and browser previews of both language versions passed.

## 5. KV guest connection lifetime

- [ ] Replace the per-call `Connection::new()` with the repository's `LazyLock<Connection>` implementation.
- [ ] Explain briefly that the connection is retained between calls to the same component instance.

Locations: EN 745–764; ZH 740–759.
Reference: `wasi_mindmap/guest-kv-store-rs/src/lib.rs`.
Reason: the shown host gives each new connection an empty `HashMap`, so the old guest discards its state after every call and always returns `None`.
Validation: replace the same key twice on one instance; the second call must return the first value.

## 6. Small corrections and metadata

- [ ] Replace `cargo expand --bin host.rs` with `cargo expand -p host-rs --bin host-rs` when referring to the repository (EN 399; ZH 392).
- [ ] Repair the malformed English link to wasmtime-py issue 309 at EN 486 by removing the extra parentheses around its URL.
- [x] Record this first pass with matching changelog entries and article metadata in both languages: version `0.3.0`, dated `2026.09.22`.
- [x] Describe the versions actually tested in the latest changelog entry, matching the dependency snippets.
- [ ] Update the metadata again when the remaining fixes are complete.

Metadata locations: EN 866 onward; ZH 860 onward.

## 7. Command-component workflow

- [ ] Replace `cargo component new/check/build`, the component metadata tables, and the generated `mod bindings;` workflow with ordinary Cargo and an inline `wit_bindgen::generate!` module.
- [ ] Use the repository's ordered WIT paths, selected host world, and `generate_all` option.
- [ ] Build with `cargo build --target wasm32-wasip2` and update artifact paths from `wasip1` to `wasip2`.
- [ ] Add the verified `wac plug` command with explicit input/output paths, followed by `wasmtime run` and expected output `result: 3`.
- [ ] Explain that the binary's `main()` supplies the `wasi:cli/run` entrypoint through the Rust target and that composition supplies the imported adder implementation.
- [ ] Clarify that custom imports may exist before composition; direct CLI execution requires those imports to be satisfied.
- [ ] Update the old Rust-guide link to the current runnable-components guide; keep the visual composition walkthrough as an alternative.

Locations: EN 512–609; ZH 505–601.
Reference: `wasi_mindmap/host-command-component/{Cargo.toml,src/main.rs,README.md}`.
Validation: follow the replacement build, composition, and run commands from the stated directory.

## 8. Python setup, paths, and loader explanation

- [ ] Use the tested member-directory `uv sync` and `uv run` workflow, including Python >=3.12 and the repository's locked dependency versions.
- [ ] Specify where binding generation and componentization run, and adjust WIT paths after entering the generated directory.
- [ ] Explain that `adder` is the output directory containing the generated `wit_world` package; keep the working `WitWorld` implementation.
- [ ] Replace the pinned `pip install "wasmtime==38.0.0"` instruction with the repository's `uv` setup.
- [ ] Document where `guest_adder_rs.wasm` must be placed for the magic loader to find it through `sys.path`, including the existing repository symlink when using that layout.
- [ ] Say the magic loader avoids a manual binding-generation step; it still generates bindings internally.
- [ ] Include the final host command, `uv run python host.py`, when following the repository workflow.
- [ ] Qualify Python component/resource support limitations by the version used in the tutorial rather than making an unversioned claim about current upstream support.

The package version is now pinned to `38.0.0` by item 1; replacing the installation workflow remains pending.

Locations: EN 223–254, 403–465, 486–487; ZH 216–247, 396–458, 479–480.
Reference: `wasi_mindmap/guest-adder-py/{README.md,pyproject.toml}` and `host-py/{README.md,pyproject.toml}`.
Validation: follow the guest and host README sequences from their respective directories.

## 9. Command-component diagram label

- [ ] Change `guest_adder_rs.wasm` to `guest_interfaced_adder_rs.wasm` in `command-component.png`, matching the interface used for composition.

The image is shared by both language versions.
Validation: inspect the rendered label for correctness and fit after editing.

## Completion checks

- [x] Apply matching changes for items 1–4 to both article languages.
- [x] Validate the changed Rust examples using the repository versions and layout before the editorial trim.
- [x] Check the changed code blocks and local section links, then preview both articles.
- [x] Keep this checklist updated as fixes are completed.
- [ ] Complete the remaining items and repeat their relevant validation before treating the full tutorial as updated.
