# myriad/tools — OS image builders (hosted Orum)

These tools build bootable images. Booting is the OS's job, so they live
here in Myriad-OS — not in Menagerie (kernel only) and not in Orum
(language only).

All are hosted Orum programs (they use fopen/fread/fwrite/fclose/fseek/
heap_alloc). No argv in hosted Orum yet, so file paths are constants
documented in each file's header. Tools with `take` imports compile via
`compile_source(src, base_dir="tools")` — `build_binary()` doesn't take
`base_dir`. Entry files use the `fn _start` + `exit(0)` trick so taken
modules' top-level statements don't run.

- `pe.orum` — PE32+ writer (EMBED design): `in.elf` (bootloader) +
  `in_kern.elf` (kernel) + `lowmem.bin` → `out.efi` with three sections:
  `.text` (loader code), `.kern` (kernel full first segment, headers
  included, so the loader reads e_entry from it), `.lowmem` (lowmem
  blob). Also patches the kernel's `__orum_heap_start/end` relocs to
  0x2000000/0x3000000. `fn pe_build(elf, kern, lowmem, efi)` for reuse.
- `lowmem.orum` — lowmem blob builder: ISR stubs (256 x 16B) + common
  handler (80B, on_irq address patched by the loader at runtime) +
  font (54 glyphs) + glyph index + scancode/HID tables + strings →
  `lowmem.bin` (7048 bytes, copied by the loader to 0x6000).
- `fat.orum` — FAT16 writer: `in.efi` (+ optional `in_readme.txt` →
  `README  TXT`) → `out.img`. Byte-identical to `pe_fat.build_uefi_disk`.
  Streams sector-by-sector (hosted heap is 1 MiB). `fn fat_build(efi,
  readme, img, total_mb, spc)` for reuse.
- `build.orum` — one-shot: `take`s lowmem+pe+fat, runs `lowmem_build` →
  `lowmem.bin` → `pe_build` → `mid.efi` → `fat_build` → `out.img`.
- `common.orum` — shared helpers, declarations only: `align_up`,
  `rd16/32/64`, `wr16/32/64`, `zero_fill`, `copy_bytes`, `read_all`.

Known constraints: max 6 args per call, module constants not visible
inside `fn` bodies (pass what fns need as params), `out` is a reserved
word, no `and`/`or`/`break`/`continue`/`match` (reserved).
