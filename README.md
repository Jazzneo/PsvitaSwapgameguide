# Detail Guide — swapping the game data in this ss-delta install

How to point the single-game `ss-delta` front end at a **different Vita game**, the way
`fs/ux0/app/PCSF00532` was swapped from Soul Sacrifice Delta to Valhalla Knights 3. Written to be
followed top to bottom next time, with a worked example at the end.

All paths are relative to the repo root `/run/media/system/game2/ss-delta`.

---

## 0. How the front end decides what to run (read this first)

This `ss-delta` build is **not** a game library — it is one game's wrapper:

- The title ids it knows are compiled into the binary as two literals:

  | Offset | String | Meaning |
  | --- | --- | --- |
  | `0x2170E83` | `PCSA00152` | US SSD title id |
  | `0x2170E8D` | `PCSF00532` | EU SSD title id |

- The importer installs the game at `fs/ux0/app/<title id>` and the launcher boots that folder.
  With our install the folder is `PCSF00532`, and the log's first line is `[run]: Title: PCSF00532`.
- Everything under that folder is just files, so **whatever sits in `fs/ux0/app/PCSF00532` is what
  runs**, regardless of what the game actually is. WK3's own title id (`PCSE00244`, from
  `sce_sys/param.sfo`) is only used later, for patches, saves and trophies.
- The **window title, `VK3.ini` name and `VK3/` data folder** come from a separate profile name/id
  pair patched into the binary (see §4). They are cosmetic/for tidiness — they do not choose the game.

So the swap is: *park the old app folder, put the new game's app folder in its place, make sure a
licence folder exists, launch.* The rest (which game it "is") comes from the new `param.sfo`.

---

## 1. What the front end needs for a title

```
fs/ux0/
├── app/PCSF00532/                 the game itself
│   ├── eboot.bin                  REQUIRED
│   ├── sce_sys/param.sfo          REQUIRED (TITLE / TITLE_ID read here)
│   ├── sce_sys/{icon0,pic0}.png   optional (menu art)
│   ├── sce_module/                optional
│   └── …game data dirs            Data/, masterPSP2/, resource/, … as the game has them
├── license/PCSF00532/             REQUIRED: at least one *.rif for the title (512 bytes)
├── user/00/                       created by the front end: saves, trophies
├── data/  music/  picture/  video/  theme/   may stay empty
└── patch/                         game updates; may stay empty
```

- **Licence**: the folder name must match the app folder the front end opens (`PCSF00532`), and it
  must contain a `.rif`. The content does not appear to be validated against the game: our
  `license/PCSF00532/` still holds SSD's `EP9000-PCSF00532_00-SOULSACDELTAEU00.rif` and WK3 booted
  with it. Add the real one in `license/<TITLE_ID>/` anyway (we added
  `license/PCSE00244/UP1023-PCSE00244_00-VALHALLAKNIGHTS3.rif`).
- **Saves**: created on demand at `fs/ux0/user/00/savedata/<TITLE_ID from param.sfo>/`
  (WK3 → `savedata/PCSE00244/`). An empty folder there is normal.
- **Trophies**: `fs/ux0/user/00/trophy/` — shared, left over from SSD; harmless.
- **appmeta/** is empty here; not required.
- The Vita **firmware** already installed in `fs` stays as it is — it is not per game.

---

## 2. Getting the new game's data into a folder

You need a **decrypted** dump (the front end runs the game's own `eboot.bin`):

- A folder dump: take its `app/<TITLE_ID>/` tree (`eboot.bin`, `sce_sys/`, data dirs).
- A `.pkg` + `.rif` (our WK3 source sits in `Valhalla Knights 3 (USA)/` as
  `app/PCSE00244/`, `<hash>.pkg`, `work.bin`, `zRIF.txt`): decrypt/unpack with the usual tooling so
  you end up with a plain `app/<TITLE_ID>/` tree. The front end's own importer
  (`./ss-delta.x86_64 --install`, or `--dump <file>`) does this interactively and builds `fs/` plus
  the AOT code for you — that is the clean route when you have the `.pkg` and the licence; the manual
  copy below is what we did for WK3.

Check the tree before copying:

```sh
ls  "<your dump>/app/"                      # expect one TITLE_ID folder
file "<your dump>/app/PCSE00244/eboot.bin"  # must be a real PS Vita ELF ("ELF 32-bit LSB ... ARM")
strings -n 8 "<your dump>/app/PCSE00244/sce_sys/param.sfo" | head -20
```

---

## 3. The swap — step by step

### 3.1 Stop the game

Kill the recorded PID. Do **not** use `pkill -f VK3.x86_64` — the pattern matches the shell running
it and kills your own command.

```sh
ps -eo pid,args | grep "[V]K3.x86_64"      # note the PID
kill <PID>
```

### 3.2 Park the current game (once)

```sh
cd /run/media/system/game2/ss-delta/fs/ux0/app
mv PCSF00532 PCSF00532.ssd-original        # SSD, already done here
# or, for a later swap:  mv PCSF00532 PCSF00532.<name>-original
```

Never delete a parked folder — it is the only copy of that game's data in here.

### 3.3 Put the new game in place

```sh
cp -a "<your dump>/app/PCSE00244" /run/media/system/game2/ss-delta/fs/ux0/app/PCSF00532
```

- Copy the **contents**, not a nested folder: `app/PCSF00532/eboot.bin` must exist, not
  `app/PCSF00532/PCSE00244/eboot.bin`.
- `cp -a` preserves the executable bit and timestamps; `eboot.bin` need not be executable but the
  data must be byte-identical.
- Big games: same filesystem copy is instant-ish; from another disk expect minutes.

### 3.4 Licence

```sh
mkdir -p /run/media/system/game2/ss-delta/fs/ux0/license/PCSE00244
cp -a "<dump>/PCSE00244.rif" /run/media/system/game2/ss-delta/fs/ux0/license/PCSE00244/
```

Also keep a `.rif` in `license/PCSF00532/` (the folder the front end opens) — one is already there.

### 3.5 Launch and read the log

```sh
cd /run/media/system/game2/ss-delta/VK3Recompiler && ./run.sh
```

`VK3/vita3k.log` (rewritten each launch) is the ground truth:

| Line | Meaning |
| --- | --- |
| `[run]: Title: PCSF00532` | which app folder the front end opened (must match the folder you filled). |
| `[load_app_impl]: Title: VALHALLA KNIGHTS 3` | `param.sfo` was parsed — this is the real game. |
| `[get_patches]: Found N patches for titleid PCSE00244` | the title id it resolved from `param.sfo`. |
| `|E| [stat_file]: Missing file …` | usually benign probing; only a problem if boot stops there. |
| abort / `module_start` hash failure | the AOT `.so` modules do not belong to this game — see §5. |

---

## 4. If the new game's folder/title must change (binary patch)

Only needed when you want the front end to open a **different folder name**, or the ini/folder/window
title to say the new game's name. Both edits are plain in-place string replacements, NUL-padded,
computer-relative (`external/boost` is missing, so rebuilding the front end is not an option).

### 4a. Which title id the front end looks for

| Offset | Bytes | Current | Length |
| --- | --- | --- | --- |
| `0x2170E83` | 9 chars + NUL | `PCSA00152` | 10 |
| `0x2170E8D` | 9 chars + NUL | `PCSF00532` | 10 |

Rules: same length or shorter, NUL-fill the rest, keep the two slots independent. Easiest is to put
the new id in the slot you use and leave the other alone. A third, purely cosmetic mention of
`PCSF00532` at `0x2170B53` is inside an `ss-delta.ini` comment — leave it.

*Verification status:* these two offsets were located in the binary's string table but have not been
tested at runtime (no second game was on hand). If the game boots to a black screen or `run` reports
the old title, the front end is still opening `PCSF00532`: put the data there instead of patching.

### 4b. Profile name / id (window title, `.ini` name, data folder)

| Offset | Slot size | Current | Meaning |
| --- | --- | --- | --- |
| `0x2170F98` | 21 bytes (20 chars + NUL) | `Valhalla Knights 3` | window title, menu label |
| `0x2170FAD` | 8 bytes (7 chars + NUL) | `VK3` | `VK3.ini`, `VK3/` data dir, `.desktop`/`.ico` stem |

The following literal `.x86_64` starts at `0x2170FB6` — do not touch it.

Do the edit with a small script rather than an editor:

```sh
cd /run/media/system/game2/ss-delta/VK3Recompiler
cp -n VK3.x86_64 VK3.x86_64.bak-$(date +%Y%m%d)    # keep one backup
python3 - <<'EOF'
p = "VK3.x86_64"
name, pid = b"My Next Game", b"NG1"      # <=20 and <=7 chars
data = bytearray(open(p, "rb").read())
def put(off, slot, s):
    assert len(s) < slot, s
    data[off:off+slot] = s + b"\0" * (slot - len(s))
put(0x2170F98, 21, name)
put(0x2170FAD, 8,  pid)
open(p, "wb").write(data)
EOF
./VK3.x86_64 --help | head -2            # title line shows the new name
```

*Verified:* patching a copy of `VK3.x86_64.bak-doublebuffer` with these two slots makes
`./binary --help` print `My Next Game (NG1)`, i.e. both slots are the right ones.

Then rename the settings/data to match the new id (`NG1`):

```sh
mv VK3.ini NG1.ini 2>/dev/null
mv VK3 NG1 2>/dev/null                   # config.yml, log, elfdumps, player2/
# keep the launchers pointing at the binary, they do not care about the id:
sed -i 's/\.\/VK3\.x86_64/.\/VK3.x86_64/' run.sh run-player2.sh   # no-op, just check
```

(If you keep the id `VK3`, nothing needs renaming.)

---

## 5. AOT modules (`*_aot_*.so`) and `recompiled.enabled`

The `ssd_aot_*.so` files in the repo root are **SSD's** recompiled guest code. They are hash-checked
against the loaded `eboot.bin`; for any other game the check fails and the front end aborts in
`module_start`. That is why every launcher passes:

```sh
./VK3.x86_64 --set recompiled.enabled=false "$@"
```

Consequences:

- The new game runs on the **JIT** (dynarmic) only — slower, but it runs.
- `[recompiled] jit = false` in the ini is fine; it is subordinate to `enabled`.
- Proper AOT for the new game requires building it (the front end's importer/`--dump` does this with a
  downloaded compiler), which is what the missing submodules/boost block us from doing by hand.
- Never leave the SSD `.so` files named so they get loaded for the new game; with
  `recompiled.enabled=false` they are ignored.

---

## 6. Verify

```sh
cd /run/media/system/game2/ss-delta/VK3Recompiler
grep -m3 -E "\[run\]: Title|load_app_impl|get_patches" VK3/vita3k.log
ps -eo pid,etime,args | grep "[V]K3.x86_64"
```

Then look at the window: the game's own logo/title screen should appear within a minute or two, at
960x544, Vulkan, 30 FPS. Check `fs/ux0/user/00/savedata/` gained a folder named after the new
`TITLE_ID` — that proves the game got past boot into its own code.

Screenshot for records (Wayland, this machine):

```sh
DISPLAY=:0 spectacle -b -n -f -o /tmp/newgame.png
```

---

## 7. Reverting to Soul Sacrifice Delta

```sh
cd /run/media/system/game2/ss-delta/fs/ux0/app
mv PCSF00532 ssd-would-be-restored      # or delete the swapped-in copy
mv PCSF00532.ssd-original PCSF00532
cd ../../../VK3Recompiler && ./run.sh
```

Optionally restore the binary and settings names:

```sh
cp VK3.x86_64.bak-doublebuffer VK3.x86_64   # original SSD profile strings
mv VK3.ini ss-delta.ini 2>/dev/null; mv VK3 ss-delta 2>/dev/null
```

The root `run-wk3.sh` and `ss-delta.x86_64` still work for the original layout.

---

## 8. Offset reference (single page)

| What | Offset | Size |
| --- | --- | --- |
| US title id literal | `0x2170E83` | 10 |
| EU title id literal | `0x2170E8D` | 10 |
| Profile display name | `0x2170F98` | 21 |
| Profile id (`ini`/folder stem) | `0x2170FAD` | 8 |
| `.x86_64` literal | `0x2170FB6` | 8 |

Binary: `VK3Recompiler/VK3.x86_64` (pristine copy before patching: `VK3.x86_64.bak-doublebuffer`).

---

## 9. Pitfalls

- **`config.yml` is regenerated every launch** from compiled defaults. Settings changed there do not
  stick; use `VK3.ini` (which is honoured) or `--set section.key=value`.
- **Per-game XML override** works and is the only way to set some GPU options:
  `VK3/config/config_<run_app_path>.xml` — for this install the path is `PCSF00532`, i.e.
  `config_PCSF00532.xml`, *not* the game's real title id. Every GPU attribute must be present in that
  node or it is reset.
- **Folder name vs title id**: `app/PCSF00532` is the folder that runs; `PCSE00244` is the identity the
  game reports. Do not "fix" the folder name and expect it to still boot — the front end looks for the
  compiled-in id.
- **Nested folder** after copying is the most common mistake; check `app/PCSF00532/eboot.bin` exists.
- **Missing licence** generally shows as the game aborting right after `load_app_impl`.
- **No saves** are created until the game writes one; `savedata/<TITLE_ID>` may be empty. Do not
  delete saves belonging to a parked game.
- **Do not `pkill -f VK3.x86_64`**; kill the PID.
- The parked game folders and `VK3.x86_64.bak-*` files are the only backups. Keep them.

---

## 10. Worked example (what was actually done)

```sh
# 1. source: extracted US WK3 dump already in the repo
ls "Valhalla Knights 3 (USA)/app/"            # PCSE00244

# 2. park SSD
cd fs/ux0/app && mv PCSF00532 PCSF00532.ssd-original

# 3. install WK3 under the name the front end boots
cp -a "/run/media/system/game2/ss-delta/Valhalla Knights 3 (USA)/app/PCSE00244" PCSF00532

# 4. licence
mkdir -p ../license/PCSE00244
cp -a "/run/media/system/game2/ss-delta/Valhalla Knights 3 (USA)"/*.rif ../license/PCSE00244/ 2>/dev/null

# 5. front end copied out of the SSD install and patched so the window/ini/folder say WK3
cp ../ss-delta.x86_64 ../VK3Recompiler/VK3.x86_64
python3 - <<'EOF'
p="/run/media/system/game2/ss-delta/VK3Recompiler/VK3.x86_64"
d=bytearray(open(p,"rb").read())
d[0x2170F98:0x2170F98+21]=b"Valhalla Knights 3"+b"\0"*3
d[0x2170FAD:0x2170FAD+8]=b"VK3"+b"\0"*5
open(p,"wb").write(d)
EOF

# 6. launchers with AOT off, then run
#    run.sh / run-player2.sh:  ./VK3.x86_64 --set recompiled.enabled=false "$@"
./run.sh
```

Result: `[run]: Title: PCSF00532`, `[load_app_impl]: Title: VALHALLA KNIGHTS 3`,
`[get_patches]: Found 0 patches for titleid PCSE00244`, title screen "VALHALLA KNIGHTS 3 / PCSE00244".
