# victory-proton

Run [VIctory](https://github.com/Marvisak/VIctory), or any fork of it, through GE-Proton using only one command.

- It keeps VIctory, or the fork you choose, up to date and builds it for Windows from Linux.
- It runs VIctory under GE-Proton's wine and vkd3d-proton.
- It sets up what Linux needs to behave like Windows here.

Your game folder is mounted **read-only** every time the game runs.

- [Quick start](#quick-start)
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
./victory-proton setup          # tools, GE-Proton, game folder, first build
victory-proton wineserver       # recommended: a faster wineserver (once per GE-Proton release)
victory-proton                  # update, build and play
```

`setup` does all of this:

1. Installs `rustup` and `cargo-xwin` into `~/.cargo`, without touching your shell profile.
2. Downloads the newest GE-Proton if you have none.
3. Asks for your game folder.
4. Clones and builds the default VIctory fork.
5. Sets up a wine prefix.
6. Links `~/.local/bin/victory-proton` to the script.

After that, plain `victory-proton` checks the repository for new commits, rebuilds if there are any, and starts the game.

The first build takes a few minutes. The first run of every new build is slow too, because the shader cache is keyed to the build and gets rebuilt.

## Requirements

- **A Linux PC with a Vulkan GPU** that has mesh shaders: AMD RDNA2+ with Mesa RADV, NVIDIA Turing+, or Intel Arc. Use a recent Mesa or driver.
- **Packages:** `git`, `curl`, `bubblewrap` and `winetricks`. Building the faster wineserver also needs a C toolchain, `autoconf`, `bison`, `flex` and `perl`.

  | Distribution | Command |
  |---|---|
  | Arch, CachyOS, Manjaro | `sudo pacman -S --needed git curl bubblewrap winetricks base-devel autoconf bison flex perl` |
  | Debian, Ubuntu | `sudo apt install git curl bubblewrap winetricks build-essential autoconf bison flex perl` |
  | Fedora | `sudo dnf install git curl bubblewrap winetricks gcc make autoconf bison flex perl` |
  | openSUSE | `sudo zypper install git curl bubblewrap winetricks gcc make autoconf bison flex perl` |

- **GE-Proton 11-13 or newer.** `victory-proton proton` installs the newest one. It is found in Steam's `compatibilitytools.d` (native or Flatpak), in `/usr/share/steam/compatibilitytools.d`, or in this tool's own folder. You don't need Steam.
- **Disk space:** besides the game, about 0.5 GB per profile (checkout and build), 0.5 GB for the wine prefix, 1 GB while building the wineserver, plus the Rust toolchain and Windows SDK (about 2 GB, shared).
- **The game folder:** the root of the dumped package, the folder with `MicrosoftGame.config` in it. The executable is read from that file.

## Choosing a VIctory build (profiles)

Several people develop VIctory in their own forks. A **profile** is one repository and branch. Each profile keeps its own checkout, build, shader cache, saves and logs, so you can switch between them freely and they never mix. Only the wine prefix is shared.

Built-in profiles:

| Profile | Repository | What it is |
|---|---|---|
| `fxceless` (default) | [fxceless/victory-fork](https://github.com/fxceless/victory-fork) `main` | fork focused on getting the world to render |
| `upstream` | [Marvisak/VIctory](https://github.com/Marvisak/VIctory) `main` | VIctory itself |
| `myokex` | [myokex/victory-amd](https://github.com/myokex/victory-amd) `victory-amd` | fork of `fxceless` with AMD fixes |
| `joshuu` | [JoshuuDev/victory-amd](https://github.com/JoshuuDev/victory-amd) `victory-amd` | fork of `myokex`; fixes a crash as the first cutscene ends |
| `copperearth` | [Copperearth/VIctory-wineless](https://github.com/Copperearth/VIctory-wineless) `main` | native Linux runtime: no wine (see [Native profiles](#native-profiles)) |

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
| `victory-proton wineserver` | build the faster wineserver for your GE-Proton (see [How it works](#how-it-works)) |
| `victory-proton vkd3d` | build or update native vkd3d-proton (native profiles only) |
| `victory-proton proton [TAG]` | install the newest GE-Proton, or the release `TAG` |
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
| `VP_PROFILE` | the one set with `profile`, else `fxceless` | profile to use |
| `VP_GAME_DIR` | the one set with `game` | game folder |
| `VP_GAME_EXE` | read from `MicrosoftGame.config` | executable inside the game folder |
| `VP_DATA` | `~/.local/share/victory-proton` | where checkouts, builds, the prefix, saves and logs go |
| `VP_CONFIG_DIR` | `~/.config/victory-proton` | where the settings go |
| `VP_SRC` | `VP_DATA/profiles/<profile>/src` | use an existing VIctory checkout instead |
| `VP_UPSTREAM` / `VP_BRANCH` | the profile's | override the repository or branch |
| `VP_PROTON` | the newest GE-Proton found | a Proton folder to use |
| `VP_WINESERVER` | the patched one, if built | path of a wineserver to use; `0` uses Proton's own |
| `VP_GPU` | detected | `amd`, `nvidia` or `intel`: decides the GPU-specific switches |
| `VP_SANDBOX` | `1` | `0` runs without bubblewrap; the game folder is then **not** protected |
| `VP_WRAP` | | a command to run wine under, for diagnostics (e.g. `perf record -e cycles:u -o /tmp/p.data --`) |

The usual wine and VIctory variables pass through as well: `VKD3D_CONFIG` (default `descriptor_heap`), `RUST_LOG` (default `info`), `WINEDEBUG`, `WINEDLLOVERRIDES` and any `VICTORY_*`. A variable you set yourself always wins over a profile's.

## How it works

```
Xbox Series game ─▶ VIctory (victory.exe) ─▶ wine ─▶ vkd3d-proton ─▶ Vulkan driver ─▶ GPU
                   built with cargo-xwin    GE-Proton  D3D12 → Vulkan    RADV / NVIDIA / ANV
```

VIctory is built for `x86_64-pc-windows-msvc` with [cargo-xwin](https://github.com/rust-cross/cargo-xwin), which fetches the Windows SDK and CRT, so no Windows machine is needed. It runs with GE-Proton's `wine` directly, without going through Steam, in a prefix of its own. The launcher takes care of everything below that Linux needs to match Windows here.

**1. A faster wineserver** (`victory-proton wineserver`; recommended, because it makes the biggest difference)

- **The problem:** wineserver keeps each process's mapped memory views in a list and walks the whole list on every map and unmap. Some games alias GPU memory one 64 KiB page at a time, which gives over 20,000 views. Each map or unmap then costs 0.5-1 ms instead of microseconds, wineserver sits at most of a CPU core, and VIctory's texture uploads stall for seconds a frame.
- **The fix:** [`wine/wineserver-view-tree.patch`](wine/wineserver-view-tree.patch) also indexes the views in a red-black tree. Each call drops to about 10 µs.

  On one machine and one scene:

  | | stock wineserver | patched |
  |---|---|---|
  | map / unmap | 450-900 / 300-500 µs | 9-12 / 10-15 µs |
  | texture refreshes per frame | 0.2-3 s | 30-57 ms |
  | wineserver CPU | 57-91 % | 3 % |

- **How it is built:** the server has to match the wine it runs with exactly. So the command fetches the sources of *your* GE-Proton release, applies GE's own wine patches and then this one, and builds only `wineserver`.
- **How it is used:** it is used automatically with that release. It is started inside the sandbox before wine, because Proton's wine ignores `WINESERVER` and always uses the server already running for the prefix.

**2. Microsoft's `d3dcompiler_47`.** VIctory compiles a few helper shaders at run time. Wine's built-in compiler can't compile them (`InterlockedMax` on `RWByteAddressBuffer`), so the GPU-driven draws are dropped and the 3D world is black. The launcher installs Microsoft's DLL with winetricks and checks it on every run.

**3. `VKD3D_CONFIG=descriptor_heap`.** Without it, vkd3d-proton caps shader-visible descriptor heaps at 1,000,000 entries. Games that ask for more fault the GPU within a minute.

**4. GE-Proton 11-13 or newer.** Older dxil-spirv generates invalid code for the first mesh-shader pipeline, and VIctory exits with code 3.

**5. GPU-specific switches.**
- **AMD:** `VICTORY_SKIP_PIPELINES=fc5e66f8f873d3bb` skips a meshlet-culling shader that page-faults the GPU minutes into the world ([fxceless/victory-fork#3](https://github.com/fxceless/victory-fork/issues/3)).
- **The forks:** they also get `VICTORY_NO_WATCHDOG=1`, so VIctory doesn't take a slow frame for a hang.
- **Late-wait switches:** some forks offer them (`VICTORY_LATE_WAIT`, `VICTORY_RELAY`, ...). They are not set, because on AMD they faulted the GPU as loading ended in 3 runs out of 4. You can still try them by setting them yourself.

**6. The sandbox.** The game runs inside [bubblewrap](https://github.com/containers/bubblewrap), which sees the whole system as usual except the game folder, which is read-only. Nothing VIctory, wine or the game does can change your dump.

## Where files go

```
~/.config/victory-proton/
  config          your VP_* settings (optional, a shell file)
  profiles        profiles you added
  profile         the default profile
  game            the game folder
~/.local/share/victory-proton/      (VP_DATA)
  prefix/         the shared wine prefix
  proton/         GE-Proton, if Steam is not installed
  wineserver/     the patched wineserver (and its ~1 GB build tree, safe to delete)
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
| Very slow scenes, or textures that never load | You are on the stock wineserver: run `victory-proton wineserver`. |
| Exit code 3 at the first 3D scene | GE-Proton is older than 11-13: run `victory-proton proton`. |
| `GPUVM fault` / `device removed` in the log | Often a fork bug. Try another profile, and make sure no late-wait switches are set. On AMD, check that `status` lists the skip. Report it to the fork with the log. |
| A crash ~45 s in, with `VKD3D_CONFIG` changed | Keep `descriptor_heap` in it. |
| `bubblewrap cannot create a sandbox` | Unprivileged user namespaces are off. On Ubuntu 23.10+, AppArmor restricts them; `sudo sysctl kernel.apparmor_restrict_unprivileged_userns=0` allows them until the next reboot. As a last resort, `VP_SANDBOX=0` runs without the read-only protection. |
| `victory.exe is running already` | Close it with `victory-proton kill`. |
| The first run after an update is slow | The shader cache is per build, so it is rebuilt once. |
| A fork doesn't build | Its tip may be broken. Try again later, or pin a commit by checking it out in `VP_DATA/profiles/<name>/src` (commits of your own stop the auto-update). |

Crash reports from the game are in `VP_DATA/profiles/<profile>/data/<title id>/crashes.log` and `.../temp/*.crash.log`.

When you report a problem to a fork, include:

- the log,
- your GPU and driver,
- your GE-Proton release,
- whether the patched wineserver was in use.

## Credits

- [VIctory](https://github.com/Marvisak/VIctory) by Marvisak and contributors, and the forks listed above, do all the real work.
- [GE-Proton](https://github.com/GloriousEggroll/proton-ge-custom), [Wine](https://www.winehq.org/), [vkd3d-proton](https://github.com/HansKristian-Work/vkd3d-proton) and [cargo-xwin](https://github.com/rust-cross/cargo-xwin).
- The wineserver patch is a change to Wine and is under Wine's license, LGPL-2.1-or-later.
