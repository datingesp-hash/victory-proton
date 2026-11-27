# victory-proton

Run [VIctory](https://github.com/Marvisak/VIctory), or any fork of it, through GE-Proton using only one command.

- It keeps VIctory, or the fork you choose, up to date and builds it for Windows from Linux.
- It runs VIctory under GE-Proton's wine and vkd3d-proton.
- It sets up what Linux needs to behave like Windows here.

Your game folder is mounted **read-only** every time the game runs.

- [Quick start](#quick-start)
- [Using another Proton](#using-another-proton)
- [Requirements](#requirements)
- [Choosing a VIctory build (profiles)](#choosing-a-victory-build-profiles)
- [Commands](#commands)
- [Configuration](#configuration)
- [How it works](#how-it-works)
- [Where files go](#where-files-go)
- [Troubleshooting](#troubleshooting)
- [Credits](#credits)

## Quick start

```sh
git clone https://github.com/datingesp-hash/victory-proton
cd victory-proton
./victory-proton setup          # tools, GE-Proton, game folder, first build, patched wineserver
victory-proton                  # update, build and play
```

`setup` does all of this:

1. Installs `rustup` and `cargo-xwin` into `~/.cargo`, without touching your shell profile.
2. Downloads the newest GE-Proton if you have none.
3. Asks for your game folder.
4. Clones and builds the default VIctory fork ([`kroeeee`](#choosing-a-victory-build-profiles)).
5. Sets up a wine prefix.
6. Builds the [faster wineserver](#how-it-works) for your GE-Proton (10-20 min, once per GE-Proton release).
7. For profiles that use them (`kroeeee`), sets up [huge pages](#how-it-works): it asks for `sudo` once, to give one small helper the `CAP_SYS_NICE` capability.
8. Links `~/.local/bin/victory-proton` to the script.

If you skip `setup`, plain `victory-proton` does the same on its first run, except the `sudo` step (`victory-proton thp` does that one).

After that, plain `victory-proton` checks the repository for new commits, rebuilds if there are any, and starts the game.

The first build takes a few minutes. The first run of every new build is slow too, because the shader cache is keyed to the build and gets rebuilt.

Warnings such as `warning: function ... is never used` while it builds come from the fork's own code. They are harmless and you can ignore them.

## Using another Proton

By default every profile runs with the newest GE-Proton you have. You can give any profile its own Proton:

```sh
victory-proton proton list                         # every Proton found on this PC
victory-proton profile proton fxceless experimental                # Steam's Proton - Experimental
victory-proton profile proton joshuu proton-cachyos-11.0-20260601-slr-x86_64_v3   # a name from the list
victory-proton profile proton myokex GE-Proton10-34               # a GE-Proton version (downloaded if missing)
victory-proton profile proton upstream ~/my-protons/my-custom-proton   # any Proton folder
victory-proton profile proton fxceless                             # show what that profile uses
victory-proton profile proton fxceless default                     # back to the newest GE-Proton
```

Things to know:

- **What counts as a Proton:** any folder with a `proton` script and a `version` file: GE-Proton, Steam's Protons, Proton-CachyOS, UMU-Proton, or one you built yourself. They are looked for in Steam's `compatibilitytools.d` and `steamapps/common` (native and Flatpak), in `/usr/share/steam/compatibilitytools.d` and in this tool's own folder. Names don't depend on upper or lower case.
- **Its own prefix:** a profile with its own Proton gets a wine prefix of its own (`~/.local/share/victory-proton/prefixes/<name>`), set up on its first run, so different wine versions never share one. Saves and caches stay per profile as always.
- **The fast wineserver is for GE-Proton only.** With any other Proton the game runs with that Proton's own wineserver, and busy scenes are about half as fast. `status` tells you which one is in use.
- **For one run, or for every profile:** `VP_PROTON=experimental victory-proton` overrides it for that run. `VP_PROTON` in `~/.config/victory-proton/config` applies to all profiles.
- For a fork you add, `VP_PROTON=...` among its variables does the same: `victory-proton profile add someone/fork NAME BRANCH VP_PROTON=GE-Proton10-34`.

## Requirements

- **A Linux PC with a Vulkan GPU** that has mesh shaders: AMD RDNA2+ with Mesa RADV, NVIDIA Turing+, or Intel Arc. Use a recent Mesa or driver.
- **Packages:** `git`, `curl`, `bubblewrap`, `winetricks`, `clang` and `lld` (for the build), a C toolchain, `autoconf`, `bison`, `flex` and `perl` (for the wineserver), and `setcap` (for huge pages).

  | Distribution | Command |
  |---|---|
  | Arch, CachyOS, Manjaro | `sudo pacman -S --needed git curl bubblewrap winetricks clang lld base-devel autoconf bison flex perl libcap` |
  | Debian, Ubuntu | `sudo apt install git curl bubblewrap winetricks clang lld build-essential autoconf bison flex perl libcap2-bin` |
  | Fedora | `sudo dnf install git curl bubblewrap winetricks clang lld gcc make autoconf bison flex perl libcap` |
  | openSUSE | `sudo zypper install git curl bubblewrap winetricks clang lld gcc make autoconf bison flex perl libcap-progs` |
- **RAM:** 32 GB is tight, 64 GB is better, and some swap (16 GB or more) helps: the process commits 30-40 GB.

- **GE-Proton 11-13 or newer.** `victory-proton proton` installs the newest one. It is found in Steam's `compatibilitytools.d` (native or Flatpak), in `/usr/share/steam/compatibilitytools.d`, or in this tool's own folder. You don't need Steam.
- **Disk space:** besides the game, about 0.5 GB per profile (checkout and build), 0.5 GB for the wine prefix, 1 GB for the wineserver's build tree (safe to delete afterwards), plus the Rust toolchain and Windows SDK (about 2 GB, shared).
- **The game folder:** the root of the dumped package, the folder with `MicrosoftGame.config` in it. The executable is read from that file.

## Choosing a VIctory build (profiles)

Several people develop VIctory in their own forks. A **profile** is one repository and branch. Each profile keeps its own checkout, build, shader cache, saves and logs, so you can switch between them freely and they never mix. Only the wine prefix is shared.

Built-in profiles:

| Profile | Repository | What it is |
|---|---|---|
| `kroeeee` (default) | [kroeeee-eeee/VIctory-Linux-Proton-Wine-7900XT-XTX](https://github.com/kroeeee-eeee/VIctory-Linux-Proton-Wine-7900XT-XTX) `main` | made for Linux/Proton on AMD RDNA 3 (RX 7900 XT/XTX, RADV); the fastest one here (see below) |
| `fxceless` | [fxceless/victory-fork](https://github.com/fxceless/victory-fork) `main` | fork focused on getting the world to render |
| `upstream` | [Marvisak/VIctory](https://github.com/Marvisak/VIctory) `main` | VIctory itself |
| `myokex` | [myokex/victory-amd](https://github.com/myokex/victory-amd) `victory-amd` | fork of `fxceless` with AMD fixes |
| `joshuu` | [JoshuuDev/victory-amd](https://github.com/JoshuuDev/victory-amd) `victory-amd` | fork of `myokex`; fixes a crash as the first cutscene ends |
| `copperearth` | [Copperearth/VIctory-wineless](https://github.com/Copperearth/VIctory-wineless) `main` | native Linux runtime: no wine (see [Native profiles](#native-profiles)) |

**`kroeeee`** runs with the settings of its own `linux/victory-proton.sh`: the fork's fast mode, its renderer ports 1, 2, 5, 6 and 7, `VICTORY_COPY_COMPARE=1`, the title's 30 fps cap off (`VICTORY_FRAME_INTERVAL_MS=0`; `33.333` puts it back), `VICTORY_ONE_PAD=0`, and `VKD3D_CONFIG` left empty, as its author runs it. With the patched wineserver and huge pages, its author and we measured about 17-19 fps in the open world on an RX 7900 XT; without either it is about half that. It was made and tested on AMD RDNA 3 only: on other GPUs you get a warning, and another profile may do better. Its [LINUX.md](https://github.com/kroeeee-eeee/VIctory-Linux-Proton-Wine-7900XT-XTX/blob/main/LINUX.md) has the details and known issues.

```sh
victory-proton profile                 # list them; * marks the one in use
victory-proton profile joshuu          # make it the default
victory-proton -p upstream             # use another one for this run only
```

**Adding a fork.** Paste its GitHub address. A `/tree/<branch>` link picks that branch; otherwise the repository's default branch is used. The profile is named after the owner unless you give it a name.

```sh
victory-proton profile add https://github.com/someone/victory-fork
victory-proton profile add https://github.com/someone/victory-fork/tree/some-branch
victory-proton profile add someone/victory-fork mytest some-branch VICTORY_SOMETHING=1
victory-proton -p https://github.com/someone/victory-fork     # add it and run it in one go
victory-proton profile remove mytest
```

Any `VAR=VALUE` after the address is set every time that profile runs. Use it for the switches a fork's README recommends. Profiles you add are kept in `~/.config/victory-proton/profiles`, one per line (`NAME URL BRANCH [VAR=VALUE ...]`), and you can edit that file by hand.

### Native profiles

Some forks run VIctory as a Linux program, with no wine and no Windows DLL (`copperearth`). The profile's `VP_NATIVE=1` tells the launcher. For these profiles:

- VIctory is built for Linux with plain `cargo build`, without cargo-xwin.
- There is no wine prefix, no `d3dcompiler_47` and no wineserver. GE-Proton is not used.
- vkd3d-proton is needed as native Linux libraries. Proton ships only its Windows DLLs, so `victory-proton vkd3d` builds it from source into `~/.local/share/victory-proton/vkd3d-proton`. This happens by itself on the first run; run the command again to update it. Set `VICTORY_VKD3D` to use one you built yourself.
- The game still runs in the same read-only sandbox, with the same per-profile folders.

Native builds also need `ninja`, `glslang` and `widl` (for vkd3d-proton; `meson` is installed into a Python venv if missing), and `SDL3` and `libpulse` at run time:

| Distribution | Command |
|---|---|
| Arch, CachyOS, Manjaro | `sudo pacman -S --needed ninja glslang wine sdl3 libpulse` |
| Debian, Ubuntu | `sudo apt install ninja-build glslang-tools wine64-tools python3-venv libsdl3-0 libpulse0` |
| Fedora | `sudo dnf install ninja-build glslang wine-devel SDL3 pulseaudio-libs` |

To mark a fork you add as native: `victory-proton profile add someone/fork NAME BRANCH VP_NATIVE=1`.

**Updating.** Every run fetches the profile's branch and fast-forwards to it. Some forks keep their changes in one commit that they rebase, which rewrites history; when that happens, the checkout follows it, as long as you haven't committed anything of your own there. If you have local commits or uncommitted changes, the checkout is left alone and you get a warning.

## Commands

| Command | What it does |
|---|---|
| `victory-proton` (or `run`) | update the profile, build it if needed, set up wine and start the game |
| `victory-proton run --no-update` | skip the update and use the current build |
| `victory-proton run -- ARGS` | pass `ARGS` on to VIctory |
| `victory-proton setup` | install the tools, pick the game, clone, build and set up wine |
| `victory-proton game` | show the game folder (and ask for it if none is set) |
| `victory-proton game pick` | choose the game folder again (file dialog, or a prompt in the terminal) |
| `victory-proton game PATH` | set the game folder |
| `victory-proton profile ...` | list, choose, add or remove profiles (see above) |
| `victory-proton -p NAME\|URL COMMAND` | run any command with another profile |
| `victory-proton wineserver` | build the faster wineserver for your GE-Proton now (a run does it by itself when it is missing; see [How it works](#how-it-works)) |
| `victory-proton thp` | set up huge pages for the profiles that use them (`kroeeee`): builds the helper and gives it `CAP_SYS_NICE` with `sudo`, once |
| `victory-proton vkd3d` | build or update native vkd3d-proton (native profiles only) |
| `victory-proton proton [TAG]` | install the newest GE-Proton, or the release `TAG` |
| `victory-proton proton list` | every Proton found on this PC |
| `victory-proton profile proton NAME [PROTON\|default]` | run a profile with another Proton (see [Using another Proton](#using-another-proton)) |
| `victory-proton update` / `build` | only update, or only build |
| `victory-proton status` | what is set up and what is missing |
| `victory-proton log [N]` | the path of the last run's log and its last `N` lines (40 by default) |
| `victory-proton kill` | close the game |
| `victory-proton victory ARGS` | run `victory.exe` with any of its own commands (`config`, `inspect`, ...) |

Only one game can run at a time. Starting a second one would stop the wineserver the first one uses.

## Configuration

Every setting is a `VP_*` variable. Set it in the environment for one run (`VP_WINESERVER=0 victory-proton`), or put it in `~/.config/victory-proton/config`, a shell file that is read on every run.

| Variable | Default | What it does |
|---|---|---|
| `VP_PROFILE` | the one set with `profile`, else `kroeeee` | profile to use |
| `VP_GAME_DIR` | the one set with `game` | game folder |
| `VP_GAME_EXE` | read from `MicrosoftGame.config` | executable inside the game folder |
| `VP_DATA` | `~/.local/share/victory-proton` | where checkouts, builds, the prefix, saves and logs go |
| `VP_CONFIG_DIR` | `~/.config/victory-proton` | where the settings go |
| `VP_SRC` | `VP_DATA/profiles/<profile>/src` | use an existing VIctory checkout instead |
| `VP_UPSTREAM` / `VP_BRANCH` | the profile's | override the repository or branch |
| `VP_PROTON` | the profile's Proton, else the newest GE-Proton | a Proton for every profile: a folder, a name from `proton list`, a version, or `experimental` |
| `VP_WINESERVER` | the patched one (built on the first run) | path of a wineserver to use; `0` uses Proton's own and does not build one |
| `VP_GPU` | detected | `amd`, `nvidia` or `intel`: decides the GPU-specific switches |
| `VP_SANDBOX` | `1` | `0` runs without bubblewrap; the game folder is then **not** protected |
| `VP_WRAP` | | a command to run wine under, for diagnostics (e.g. `perf record -e cycles:u -o /tmp/p.data --`) |

The usual wine and VIctory variables pass through as well: `VKD3D_CONFIG` (default `descriptor_heap`, unless the profile sets it), `RUST_LOG` (default `info`), `WINEDEBUG`, `WINEDLLOVERRIDES` and any `VICTORY_*` (`VICTORY_THP=0` turns huge pages off). A variable you set yourself always wins over a profile's, and a profile's over the defaults.

## How it works

```
Xbox Series game ─▶ VIctory (victory.exe) ─▶ wine ─▶ vkd3d-proton ─▶ Vulkan driver ─▶ GPU
                   built with cargo-xwin    GE-Proton  D3D12 → Vulkan    RADV / NVIDIA / ANV
```

VIctory is built for `x86_64-pc-windows-msvc` with [cargo-xwin](https://github.com/rust-cross/cargo-xwin), which fetches the Windows SDK and CRT, so no Windows machine is needed. It runs with GE-Proton's `wine` directly, without going through Steam, in a prefix of its own. The launcher takes care of everything below that Linux needs to match Windows here.

**1. A faster wineserver** (built by itself on the first run, once per GE-Proton release; it makes the biggest difference)

- **The problem:** wineserver keeps each process's mapped memory views in a list and walks the whole list on every map and unmap. Some games alias GPU memory one 64 KiB page at a time, which gives over 20,000 views. Each map or unmap then costs 0.5-1 ms instead of microseconds, wineserver sits at most of a CPU core, and VIctory's texture uploads stall for seconds a frame.
- **The fix:** [`wine/wineserver-view-tree.patch`](wine/wineserver-view-tree.patch) also indexes the views in a red-black tree. Each call drops to about 10 µs.

  With `kroeeee` on an RX 7900 XT and a Ryzen 9 9900X, GE-Proton11-14, the same scene:

  | | stock wineserver | patched |
  |---|---|---|
  | frames per second | 8.2-8.4 | 17.0-18.5 |
  | waiting for the write watch, per frame | 17-26 ms | 4.6 ms |
  | wineserver's share of CPU samples | 11-16 % | about 1 % |
  | launch to world | 112-120 s | 43-60 s |

  If two views ever share a base address, the server stops trusting the tree, goes back to the list and says so once in the log (`wineserver view tree: ... falling back to the list`).

- **How it is built:** the server has to match the wine it runs with exactly. So the command fetches the sources of *your* GE-Proton release, applies GE's own wine patches and then this one, and builds only `wineserver`.
- **How it is used:** wine always runs the wineserver in its own `bin/` folder and ignores `WINESERVER`. So the game runs with a copy of your GE-Proton, `VP_DATA/proton-vt/<release>-vt`, made of hard links (next to no disk space) except `files/bin/wineserver`, which is the patched one. Your GE-Proton itself is never changed. If the build fails, the game runs with the stock server and you get a warning.

**2. Huge pages** (profiles that ship `linux/thp/`, such as `kroeeee`). Renderer port 1 hands a ~185 MB heap of the title to the AMD kernel driver as a user pointer, and the driver checks it page by page on every submission. With 4 KiB pages that keeps one CPU core busy and the GPU waiting; as 2 MiB huge pages the check is 512 times shorter: about 9 → 18 fps in the same scene. Every 15 s while the game runs, the profile's small helper collapses those ranges (read from the log) into huge pages from outside the process (`process_madvise(MADV_COLLAPSE)`, Linux 6.1+). Their contents don't change. The helper needs one capability, `CAP_SYS_NICE` (not root), which `victory-proton thp` gives it with `sudo` once; it is rebuilt, and asks again, only when the fork changes it. Results are in a `.thp` file next to the run's log (`88 collapsed, 0 not`).

**3. Microsoft's `d3dcompiler_47`.** VIctory compiles a few helper shaders at run time. Wine's built-in compiler can't compile them (`InterlockedMax` on `RWByteAddressBuffer`), so the GPU-driven draws are dropped and the 3D world is black. The launcher installs Microsoft's DLL with winetricks and checks it on every run.

**4. `VKD3D_CONFIG=descriptor_heap`** (for profiles that don't set it themselves). Without it, vkd3d-proton caps shader-visible descriptor heaps at 1,000,000 entries. Games that ask for more fault the GPU within a minute.

**5. GE-Proton 11-13 or newer.** Older dxil-spirv generates invalid code for the first mesh-shader pipeline, and VIctory exits with code 3.

**6. GPU-specific switches.**
- **AMD:** unless the profile sets its own, `VICTORY_SKIP_PIPELINES=fc5e66f8f873d3bb` skips a meshlet-culling shader that page-faults the GPU minutes into the world ([fxceless/victory-fork#3](https://github.com/fxceless/victory-fork/issues/3)).
- **The older forks:** they also get `VICTORY_NO_WATCHDOG=1`, so VIctory doesn't take a slow frame for a hang.
- **Late-wait switches:** some forks offer them (`VICTORY_LATE_WAIT`, `VICTORY_RELAY`, ...). They are not set for the older forks, because on AMD they faulted the GPU as loading ended in 3 runs out of 4. `kroeeee` sets the ones its tree was tuned with.

**7. The sandbox.** The game runs inside [bubblewrap](https://github.com/containers/bubblewrap), which sees the whole system as usual except the game folder, which is read-only. Nothing VIctory, wine or the game does can change your dump.

## Where files go

```
~/.config/victory-proton/
  config          your VP_* settings (optional, a shell file)
  profiles        profiles you added
  profile         the default profile
  game            the game folder
  protons         the Proton chosen for each profile
~/.local/share/victory-proton/      (VP_DATA)
  prefix/         the shared wine prefix (profiles on the default Proton)
  prefixes/<name>/  a prefix for each Proton chosen for a profile
  proton/         GE-Proton, if Steam is not installed
  wineserver/     the patched wineserver (and its ~1 GB build tree, safe to delete)
  proton-vt/      GE-Proton copies (hard links) with the patched wineserver
  thp/            the huge-page helper
  latest.log      the last run's log, of any profile
  profiles/<name>/
    src/          the fork's checkout and build
    cache/        shader and image caches
    data/         saves, storage and crash reports
    logs/         one log per run
```

To remove everything, delete those two folders and `~/.local/bin/victory-proton`.

## Troubleshooting

Start with `victory-proton status`, and look at the log with `victory-proton log`.

| What you see | Likely cause and fix |
|---|---|
| The world is black; only subtitles or credits show | Wine's `d3dcompiler_47` is in use. The next run installs Microsoft's; check `status`. |
| Very slow scenes, or textures that never load | You are on the stock wineserver (`status` says so; the build may have failed for a missing package): run `victory-proton wineserver` and read its error. |
| About half the frame rate you expected with `kroeeee` | Huge pages are off: run `victory-proton thp`, and check the `.thp` file next to the log. |
| The game steers on its own | A device that pretends to be a joystick (drawing tablet, analog keyboard): list it in `SDL_GAMECONTROLLER_IGNORE_DEVICES` and `SDL_JOYSTICK_IGNORE_DEVICES` (`0xVID/0xPID,...`). |
| Exit code 3 at the first 3D scene | GE-Proton is older than 11-13: run `victory-proton proton`. |
| `GPUVM fault` / `device removed` in the log | Often a fork bug. Try another profile. With the older forks, make sure no late-wait switches are set and, on AMD, that `status` lists the skip. Report it to the fork with the log. |
| Don't set `RADV_DEBUG=hang` | It turns on synchronised shaders and makes the GPU the bottleneck (about 4 fps). |
| A crash ~45 s in, with `VKD3D_CONFIG` changed | With the older forks, keep `descriptor_heap` in it. |
| `bubblewrap cannot create a sandbox` | Unprivileged user namespaces are off. On Ubuntu 23.10+, AppArmor restricts them; `sudo sysctl kernel.apparmor_restrict_unprivileged_userns=0` allows them until the next reboot. As a last resort, `VP_SANDBOX=0` runs without the read-only protection. |
| `victory.exe is running already` | Close it with `victory-proton kill`. |
| The first run after an update is slow | The shader cache is per build, so it is rebuilt once. |
| A fork doesn't build | Its tip may be broken. Try again later, or pin a commit by checking it out in `VP_DATA/profiles/<name>/src` (commits of your own stop the auto-update). |

Crash reports from the game are in `VP_DATA/profiles/<profile>/data/<title id>/crashes.log` and `.../temp/*.crash.log`.

When you report a problem to a fork, include:

- the log,
- your GPU and driver,
- your GE-Proton release,
- whether the patched wineserver and huge pages were in use (both in `status`).

## Credits

- [VIctory](https://github.com/Marvisak/VIctory) by Marvisak and contributors, and the forks listed above, do all the real work.
- [GE-Proton](https://github.com/GloriousEggroll/proton-ge-custom), [Wine](https://www.winehq.org/), [vkd3d-proton](https://github.com/HansKristian-Work/vkd3d-proton) and [cargo-xwin](https://github.com/rust-cross/cargo-xwin).
- The wineserver patch is a change to Wine and is under Wine's license, LGPL-2.1-or-later.
