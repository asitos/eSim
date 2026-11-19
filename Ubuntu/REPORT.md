# eSim 2.5 on Ubuntu 25.04 – Installation & Debugging Report

**Contributor:** Ashutosh Parashar ([@asitos](https://github.com/asitos))
**Target Environment:** Ubuntu 25.04 (Plucky Puffin)
**Branch:** `installers`

---

## Background & Previous Work

Earlier this year, PR #414 (commit `f4cede1a` by `@muaviz`) added initial scaffolding for Ubuntu 25.04 by creating `install-eSim-scripts/install-eSim-25.04.sh` and outlining upcoming dependency targets (such as pinning LLVM to version 18, switching to GTK3 Canberra, and adding the KiCad 8 PPA).

However, live installation on a fresh Ubuntu 25.04 VirtualBox VM revealed that the codebase still contained blocking issues: merge syntax errors, misplaced release assets, shell exit bugs, and uncommitted updates to dependency archives.

Below is a record of the issues investigated, traced through git history, and resolved so far.

---

## Summary of Issues Worked on So Far

| # | Issue | Component | Root Cause | Status & Commit |
|---|---|---|---|:---:|
| 1 | `syntax error near unexpected token '}'` | `Ubuntu/install-eSim.sh` | Unclosed `if` block from merge conflict resolution | **Fixed** (`d19714a1`) |
| 2 | Misplaced binary archives in `MacOS/` | Repository layout | `kicadLibrary.tar.xz`, `nghdl.zip`, `sky130_fd_pr.tar.xz` committed into `MacOS/` | **Fixed** (`58b7a219`, `d088fc94`, `c9a4fedc`) |
| 3 | Premature installer termination after KiCad 8 check | `install-eSim-25.04.sh` | `installKicad()` called `exit 0` instead of `return 0` | **Fixed** (`fc2248c5`) |
| 4 | Symbol table copied to legacy KiCad 6 path | `install-eSim-25.04.sh` | Hardcoded `~/.config/kicad/6.0` instead of `8.0` | **Fixed** |
| 5 | `Unsupported Ubuntu version: 25.04 ()` in NGHDL | `nghdl/install-nghdl.sh` | Missing 25.04 routing; GTK2 EOL and LLVM 20 API breakages | **Fixed** (`2dff020a`, `462c91ad`, `424d1c28`) |
| 6 | `cp: cannot stat 'images/logo.png': No such file` | Repository layout | UI branding assets committed under `MacOS/images/` | **Fixed** (`280df64b`) |
| 7 | `esim` launcher fails: `cd: .../src/frontEnd: No such file` | Repository layout & runtime | `src/`, `Examples/`, and sub-libraries trapped in `MacOS/` | **Fixed** |

---

## Detailed Breakdown of Diagnosed & Resolved Issues

### 1. Line 70 Syntax Error in `install-eSim.sh`

* **Symptom:**
  Running `./Ubuntu/install-eSim.sh --install` crashed immediately:
  ```text
  ./Ubuntu/install-eSim.sh: line 70: syntax error near unexpected token `}'
  ./Ubuntu/install-eSim.sh: line 70: `}'
  ```
* **Git Forensics (`git blame`):**
  Traced with `git blame -L 55,75 a77d6b90 Ubuntu/install-eSim.sh` to merge commit `52d77f6d` (merging PR #414 into `installers`). During merge conflict resolution, `run_version_script()` was inserted without closing the `if [[ -f "$SCRIPT" ]]; then` block with `fi`, followed by 7 orphaned lines from an old `createConfigFile` block.
* **Fix Applied (Commit `d19714a1`):**
  Closed the conditional with `fi`, added exit code forwarding (`exit $?`), added an error message if the targeted script is missing, and deleted the orphaned lines. Verified with `bash -n Ubuntu/install-eSim.sh`.

---

### 2. Misplaced Archives in `MacOS/` Folder

* **Symptom:**
  After the dispatcher fix, KiCad installed, but extracting symbol libraries failed:
  ```text
  tar (child): library/kicadLibrary.tar.xz: Cannot open: No such file or directory
  ```
* **Git Forensics (`git log`):**
  Traced with `git log --diff-filter=A --summary -- MacOS/library/kicadLibrary.tar.xz` to commit `29a8e8e6` (*"Added macos support installing folder"*). Shared binary assets (`kicadLibrary.tar.xz`, `nghdl.zip`, `sky130_fd_pr.tar.xz`) were committed under `MacOS/` rather than the repository root. Auditing `Fedora/install-eSim_fedora.sh`, `Ubuntu/install-eSim-scripts/*.sh`, and `MacOS/build_esim.sh` confirmed that all platforms expect these files relative to root.
* **Fix Applied (Commit `58b7a219`):**
  Moved `kicadLibrary.tar.xz` to `library/kicadLibrary.tar.xz` and staged moves for `nghdl.zip` and `sky130_fd_pr.tar.xz`.

---

### 3. Premature Installer Termination in `installKicad()`

* **Symptom:**
  Re-running the installer on a machine where KiCad 8 was already present printed:
  ```text
  Installing KiCad...........................
  Ubuntu 25.04 detected.
  KiCad 8.0 is already installed.
  ```
  ...and then abruptly exited with code 0 without running `copyKicadLibrary`, `installNghdl`, or `installSky130Pdk`.
* **Git Forensics (`git log -S`):**
  Traced with `git log -S "KiCad 8.0 is already installed"` to commit `5088e227` (PR #323), copied into `install-eSim-25.04.sh` in PR #414. Calling `exit 0` inside a shell function terminates the entire shell process rather than returning to the caller.
* **Fix Applied (Commit `fc2248c5`):**
  Replaced `exit 0` with `return 0` so the installer acknowledges existing KiCad installations and continues with the rest of the setup.

---

### 4. KiCad Symbol Table Destination Path (6.0 vs 8.0)

* **Symptom:**
  During installation on Ubuntu 25.04 with KiCad 8.0:
  ```text
  .config/kicad/6.0 does not exist
  symbol table copied in the directory
  ```
* **Root Cause Analysis:**
  `installKicad()` installs KiCad 8 (`kicad=8.0.8+dfsg-1`), but `copyKicadLibrary()` hardcoded `~/.config/kicad/6.0/sym-lib-table`. KiCad 8 ignores configuration in `6.0` and expects custom symbol tables in `~/.config/kicad/8.0/sym-lib-table`. Without this file, launching the KiCad schematic editor for the first time prompts the user to initialize an empty table or choose defaults, omitting eSim's custom symbol libraries (`eSim_Devices`, `eSim_Subckt`, `eSim_SKY130`, etc.).
* **Fix Applied:**
  Updated `copyKicadLibrary()` in `install-eSim-25.04.sh` to iterate across both `8.0` and `6.0` config directories, ensuring directory creation and copying `sym-lib-table` to `~/.config/kicad/8.0/`. Also updated the `--uninstall` cleanup routine to purge `~/.config/kicad/8.0/`.

---

### 5. NGHDL Ubuntu 25.04 Dispatch & Toolchain Support

* **Symptom:**
  After unpacking `nghdl.zip`, the internal script failed:
  ```text
  Installing NGHDL...........................
  Archive:  nghdl.zip
    inflating: nghdl/install-nghdl.sh
  ...
  Detected Ubuntu Version:
  Unsupported Ubuntu version: 25.04 ()
  ```
* **Git Forensics (`git log`):**
  PR #414 documented intent to update NGHDL for Ubuntu 25.04, but inspecting commit `f4cede1a` confirmed that `nghdl.zip` was never modified or committed to Git. The archive retained its older `install-nghdl.sh` router which only supported releases up to `24.04`.

* **Technical Cause & Upstream Citations:**
  1. **GTK2 Canberra EOL:** Debian and Ubuntu 25.04 removed `libcanberra-gtk-module` because GTK 2 is deprecated ([Ubuntu Package Archive: `libcanberra-gtk-module`](https://packages.ubuntu.com/search?keywords=libcanberra-gtk-module), [Debian Package Tracker: `libcanberra`](https://tracker.debian.org/pkg/libcanberra)). Attempting to install it causes `apt` to fail with `no installation candidate`. The modern replacement used by GNOME 48 and GTK3 desktops is `libcanberra-gtk3-module`.
  2. **LLVM 18 Compatibility Ceiling:** Ubuntu 25.04 defaults to `llvm-20.1` (`llvm-20`). The bundled GHDL release ([`ghdl-4.1.0`](https://github.com/ghdl/ghdl/releases/tag/v4.1.0)) interacts directly with LLVM's internal C++ compiler APIs (`src/ortho/llvm/`). Because LLVM makes breaking C++ API changes across major versions without backward stability guarantees ([GHDL Issue #2711: LLVM 19/20 build failures](https://github.com/ghdl/ghdl/issues/2711)), GHDL 4.1.0 fails during `make` against LLVM 20. As documented in the [GHDL Building Guide](https://ghdl.github.io/ghdl/getting/building.html#llvm-backend), GHDL 4.1.0 strictly requires LLVM $\le$ 18. Using Ubuntu's co-installable `llvm-18-dev` via `/usr/bin/llvm-config-18` allows it to compile cleanly without patching GHDL source code.

* **Fix Applied (Commit `2dff020a`):**
  1. Updated `nghdl/install-nghdl.sh` to add `"25.04"` routing to `install-nghdl-25.04.sh`:
     ```diff
     --- a/nghdl/install-nghdl.sh
     +++ b/nghdl/install-nghdl.sh
     @@ -46,6 +46,9 @@ run_version_script() {
              "24.04")
                  SCRIPT="$SCRIPT_DIR/install-nghdl-24.04.sh"
                  ;;
     +        "25.04")
     +            SCRIPT="$SCRIPT_DIR/install-nghdl-25.04.sh"
     +            ;;
     ```
  2. Created `nghdl/install-nghdl-scripts/install-nghdl-25.04.sh`:
     * Switched package dependency from `libcanberra-gtk-module` to `libcanberra-gtk3-module`.
     * Explicitly installed the co-installable `llvm-18 llvm-18-dev clang-18` toolchain instead of generic `llvm`.
     * Configured GHDL with `--with-llvm-config=/usr/bin/llvm-config-18`.
  3. Re-bundled clean archive into `nghdl.zip`.

---

### 6. Misplaced UI Branding Assets (`images/logo.png`)

* **Symptom:**
  Toward the end of the installation script:
  ```text
  'esim-start.sh' -> '/usr/bin/esim'
  'esim.desktop' -> '/usr/share/applications/esim.desktop'
  'esim.desktop' -> '/home/asitos/Desktop/esim.desktop'
  cp: cannot stat 'images/logo.png': No such file or directory

  Error! Kindly resolve above error(s) and try again.
  Aborting Installation...
  ```
* **Git Forensics (`git log`):**
  Traced with `git log --diff-filter=A --summary -- MacOS/images/logo.png` to commit `29a8e8e6`. The entire `images/` directory had been committed inside `MacOS/images/`.
* **Fix Applied (Commit `280df64b`):**
  Moved `MacOS/images/` to repository root (`images/`).

---

### 7. Runtime Launcher Failure: Missing `src/frontEnd/Application.py` & Sub-Libraries

* **Symptom:**
  After installation completed, launching eSim from terminal failed:
  ```text
  $ esim
  /usr/bin/esim: line 2: cd: /home/asitos/Projects/eSim/eSim/src/frontEnd: No such file or directory
  python3: can't open file '/home/asitos/Projects/eSim/eSim/Application.py': [Errno 2] No such file or directory
  ```
* **Root Cause Analysis:**
  The launch script generated by `createDesktopStartScript()` in `/usr/bin/esim` executes:
  ```bash
  cd $eSim_Home/src/frontEnd
  source $config_dir/env/bin/activate
  python3 Application.py
  ```
  In commit `29a8e8e6`, the entire eSim application codebase (`src/`), example projects (`Examples/`), and component model libraries (`library/browser`, `deviceModelLibrary`, `modelParamXML`, `ngspicetoModelica`, `SubcircuitLibrary`, `tlv`) were placed inside `MacOS/` rather than at the repository root. All runtime dispatchers, `Appconfig.py`, `KicadtoNgspice.py`, and the start script require these paths at the root.
* **Fix Applied:**
  Relocated `src/`, `Examples/`, and all `library/*` subdirectories from `MacOS/` back to the repository root via `git mv`. `MacOS/` now retains strictly macOS packaging scripts (`build_esim.sh`, `bundle_esim.sh`, `create_pkg.sh`, `eSim.spec`, `scripts/`).
