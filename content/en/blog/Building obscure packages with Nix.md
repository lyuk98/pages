---
tags:
  - CMake
  - Nix
  - Rust
date: '2025-10-28'
title: Building obscure packages with Nix
description: It is understandable that not every software on the internet is available on any given Linux distribution, but it is sometimes disappointing when something I have been looking for is missing.
---

It is understandable that not every software on the internet is available on any given Linux distribution, but it is sometimes disappointing when something I have been looking for is missing.

# Introduction

When Arch Linux was my daily driver, [Flatpak](https://flatpak.org/ "Flatpak—the future of application distribution") (with [Flathub](https://flathub.org "Flathub - Apps for Linux")) and [Arch User Repository (AUR)](https://aur.archlinux.org/ "AUR (en) - Home") provided enough packages to satisfy my needs. However, I have since [switched to NixOS](https://lyuk98.com/62055/switching-from-arch-linux-to-nixos "Switching from Arch Linux to NixOS") and almost ditched Flatpak from my system. Even still, there were two things I could try:

1. Searching the [Nix User Repository (NUR)](https://nur.nix-community.org/ "Packages search for NUR")
2. Finding someone else who packaged the software

I recently ended up doing both while switching from Firefox to [Zen Browser](https://zen-browser.app/ "Zen Browser"). I first found that "someone else" and [used their flake](https://github.com/lyuk98/nixos-config/blob/d11125d54ef2b30e2afb95bd7f68d101c16a35a3/flake.nix#L81-L88 "nixos-config/flake.nix at d11125d54ef2b30e2afb95bd7f68d101c16a35a3 · lyuk98/nixos-config"):

```nix
# Zen Browser
zen-browser = {
  url = "github:0xc000022070/zen-browser-flake";
  inputs = {
    nixpkgs.follows = "nixpkgs";
    home-manager.follows = "home-manager";
  };
};
```

...and [used NUR](https://github.com/lyuk98/nixos-config/blob/d11125d54ef2b30e2afb95bd7f68d101c16a35a3/home/lyuk98/common/applications/network/zen-browser.nix#L85-L92 "nixos-config/home/lyuk98/common/applications/network/zen-browser.nix at d11125d54ef2b30e2afb95bd7f68d101c16a35a3 · lyuk98/nixos-config") to declare the browser's extensions:

```nix
# Manage browser extensions
ExtensionSettings =
  let
    # List of packages to allow in private browsing
    allowInPrivateBrowsing = with pkgs.nur.repos.rycee.firefox-addons; [
      ublock-origin
    ];

    # Declare extension policy from given Firefox addon packages
    mkExtensionSettings = (
      extensions:
      builtins.listToAttrs (
        builtins.map (extension: {
          name = extension.addonId;
          value =
            let
              uuid = builtins.elemAt (builtins.attrNames (builtins.readDir "${extension}/share/mozilla/extensions")) 0;
            in
            {
              install_url = "file://${extension}/share/mozilla/extensions/${uuid}/${extension.addonId}.xpi";
              installation_mode = "force_installed";
              default_area = "menupanel";

              private_browsing = builtins.elem extension allowInPrivateBrowsing;
            };
        }) extensions
      )
    );
  in
  mkExtensionSettings (
    with pkgs.nur.repos.rycee.firefox-addons;
    [
      proton-pass
      ublock-origin
    ]
  );
```

However, with the uncommon setup that is NixOS, even more so compared to Arch Linux, I eventually came across ones that I could not apply either of those workarounds to. With no other ways left, the next step was to package them myself.

The packages I wanted to use were written to [my repository](https://github.com/lyuk98/nixos-config "lyuk98/nixos-config: NixOS Configurations") for personal NixOS configurations. My goal was to eventually submit them to NUR, but I did not feel ready to do so at the time I was working on this.

# Building Friction

[![Logo of Friction](https://images.lyuk98.com/b73e0c37-9351-4bce-898e-47692af11597.svg)](https://friction.graphics/ "Friction")

> Friction is a powerful and versatile motion graphics application that allows you to create vector and raster animations for web and video.

[Friction](https://friction.graphics/ "Friction") is provided as an AppImage, which I can run within NixOS.

```
[lyuk98@framework:~/Downloads]$ nix shell nixpkgs#appimage-run
[lyuk98@framework:~/Downloads]$ appimage-run Friction-1.0.0-rc.2-x86_64.AppImage
```

However, it was not what I wanted to do. To build the software myself, I referred to [its build instructions](https://friction.graphics/documentation/source-linux.html "Build on Linux") throughout the process.

## Defining the package

Within `packages/default.nix`, I defined the package I was to work on:

```nix
{
  pkgs ? import <nixpkgs> { },
  ...
}:
{
  # personal programs

  # Packages from the internet
  friction = pkgs.libsForQt5.callPackage ./friction { };
}
```

Creation of a derivation at `packages/friction/default.nix` followed. One of the requirements was `clang`, so I used its `stdenv`.

```nix
{
  clangStdenv,
  lib,
}:
clangStdenv.mkDerivation (finalAttrs: {
  pname = "friction";
  version = "1.0.0-rc.2";

  meta = {
    description = "Powerful and versatile motion graphics application";
    longDescription = ''
      Friction is a powerful and versatile motion graphics application that allows you to create
      vector and raster animations for web and video.
    '';
    homepage = "https://friction.graphics/";
    changelog = "https://friction.graphics/releases/friction-100-rc2.html";
    license = lib.licenses.gpl3;
    mainProgram = "friction";
  };
})
```

I then defined where to get the source from. Since I had to get its submodules as well, `fetchSubmodules` was set to `true`.

```nix
{
  clangStdenv,
  fetchFromGitHub,
  lib,
}:
clangStdenv.mkDerivation (finalAttrs: {
  # package metadata

  src = fetchFromGitHub {
    owner = "friction2d";
    repo = "friction";
    tag = "v${finalAttrs.version}";
    fetchSubmodules = true;
    hash = lib.fakeHash;
  };
})
```

`lib.fakeHash` was used to get the hash of the source. By intentionally failing to build, I was able to get the expected value:

```
[lyuk98@framework:~/nixos-config]$ nix build .#friction
error: hash mismatch in fixed-output derivation '/nix/store/myjnkbm14q5mddfxi6piahnwh3klk0rp-source.drv':
         specified: sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=
            got:    sha256-tmxzEfOy+eCe7K4Rv+bFNk0t3aD1n4iqAroB1li9vVM=
error: Cannot build '/nix/store/b2lkqgqpfdza380xilb9dg6lzc3b9lc3-friction-1.0.0-rc.2.drv'.
       Reason: 1 dependency failed.
       Output paths:
         /nix/store/70a9i3220449c47nsfmk8ab9d2m4i755-friction-1.0.0-rc.2
```

...which I applied to my package definition:

```diff
-    hash = lib.fakeHash;
+    hash = "sha256-tmxzEfOy+eCe7K4Rv+bFNk0t3aD1n4iqAroB1li9vVM=";
```

Including dependencies was next. Friction is built using CMake, so I included the necessary flags as well.

```nix
{
  clang,
  clangStdenv,
  cmake,
  expat,
  fetchFromGitHub,
  ffmpeg,
  fontconfig,
  freetype,
  harfbuzz,
  icu,
  lib,
  libjpeg,
  libpng,
  libsysprof-capture,
  libunwind,
  libwebp,
  ninja,
  pkg-config,
  python3,
  qscintilla,
  qt5,
  qtbase,
  wrapQtAppsHook,
  zlib,
}:
let
  harfbuzz-icu = harfbuzz.override { withIcu = true; };
  qtVersion = lib.versions.major qtbase.version;
in
clangStdenv.mkDerivation (finalAttrs: {
  # package metadata and source definition

  buildInputs = [
    expat
    ffmpeg
    fontconfig
    freetype
    harfbuzz-icu
    icu
    libjpeg
    libpng
    libsysprof-capture
    libunwind
    libwebp
    qscintilla
    qt5.qtdeclarative
    qt5.qtmultimedia
    qtbase
    zlib
  ];
  nativeBuildInputs = [
    clang
    cmake
    ninja
    pkg-config
    python3
    wrapQtAppsHook
  ];

  cmakeFlags = [
    "-DCMAKE_BUILD_TYPE=Release"

    "-DQSCINTILLA_INCLUDE_DIRS=${qscintilla}/include"
    "-DQSCINTILLA_LIBRARIES_DIRS=${qscintilla}/lib"
    "-DQSCINTILLA_LIBRARIES=qscintilla2_qt${qtVersion}"
  ];
})
```

However, there was a problem with this setup.

## Customising FFmpeg

One of the requirements is FFmpeg, but 4.2 was the recommended version.

> - ffmpeg 4.2 recommended, [will not build on 7+](https://github.com/friction2d/friction/issues/135)
> 	- libavformat
> 	- libavcodec
> 	- libavutil
> 	- libswscale
> 	- libswresample

Some previous major versions of FFmpeg are maintained at Nixpkgs, but the closest one I could get was FFmpeg 4.4, which was still not ideal.

```
[lyuk98@framework:~]$ nix repl
nix-repl> pkgs = import <nixpkgs> {}
nix-repl> lib = pkgs.lib
nix-repl> lib.filterAttrs (pkg: _: lib.hasPrefix "ffmpeg_4" pkg) pkgs
{
  ffmpeg_4 = «derivation /nix/store/c70gm8zv6q802wz38lfij2xdqlp7bnvl-ffmpeg-4.4.6.drv»;
  ffmpeg_4-full = «derivation /nix/store/pz9c70v7bldj672f9c0p21nipn20c5ys-ffmpeg-full-4.4.6.drv»;
  ffmpeg_4-headless = «derivation /nix/store/hjhimbk0r2bfcvmgnx63pa3yr2wmi5sn-ffmpeg-headless-4.4.6.drv»;
}
```

Therefore, I decided to create a derivation for the dependency as well. I did not want to deal with writing one for something as complex as FFmpeg, though, so I reused the existing derivation and overrode some attributes. After a lot of trial and errors, I ended up with the following:

```nix
{
  # parameters
}:
let
  ffmpeg_4-headless =
    (ffmpeg.override {
      # Use latest version of FFmpeg 4.2
      version = "4.2.11";
      hash = "sha256-JQ1KWH09T7E2/TOGZG/okedOUQIO9bzJVwvnU7VL+b0=";

      # Disable unneeded dependencies from a headless setup
      ffmpegVariant = "headless";
      withHeadlessDeps = false;

      # Build only what is necessary
      buildAvcodec = true;
      buildAvdevice = true;
      buildAvfilter = true;
      buildAvformat = true;
      buildAvutil = true;
      buildSwresample = true;
      buildSwscale = true;

      withHardcodedTables = true;
      withNetwork = true;
      withPixelutils = true;
      withSafeBitstreamReader = true;

      buildFfmpeg = true;
    }).overrideAttrs
      (previousAttrs: {
        # Remove incompatible patch
        patches = builtins.filter (
          patch: patch.name != "unbreak-svt-av1-3.0.0.patch"
        ) previousAttrs.patches;

        # Remove nonexistent flags
        configureFlags =
          let
            nonexistentFlags = [
              "--disable-librav1e"
              "--disable-librist"
              "--disable-libsvtav1"
              "--disable-libuavs3d"
              "--disable-vulkan"
            ];
          in
          lib.lists.subtractLists nonexistentFlags previousAttrs.configureFlags;
      });

	# harfbuzz-icu and qtVersion
in
clangStdenv.mkDerivation (finalAttrs: {
  # package metadata and source definition

  buildInputs = [
    # build inputs except ffmpeg
    ffmpeg_4-headless
  ];
  nativeBuildInputs = [
    # build inputs
  ];

  cmakeFlags = [
    "-DCMAKE_BUILD_TYPE=Release"

    "-DQSCINTILLA_INCLUDE_DIRS=${qscintilla}/include"
    "-DQSCINTILLA_LIBRARIES_DIRS=${qscintilla}/lib"
    "-DQSCINTILLA_LIBRARIES=qscintilla2_qt${qtVersion}"
  ];
})
```

## Building GN

At this point, building Friction fails after being unable to locate a file:

```
[lyuk98@framework:~/nixos-config]$ nix build .#friction
error: Cannot build '/nix/store/lrni0ajy0niyiaqvp7697mj4r4n6a67m-friction-1.0.0-rc.2.drv'.
       Reason: builder failed with exit code 127.
       Output paths:
         /nix/store/kvnihck7w0l0cma18528vkg2mv16imzm-friction-1.0.0-rc.2
       Last 25 log lines:
       > [4/570] Performing patch step for 'Engine'
       > [4/570] Performing configure step for 'Engine'
       > sh: line 1: /build/source/src/engine/skia/bin/gn: cannot execute: required file not found
       > FAILED: [code=127] src/engine/Engine-prefix/src/Engine-stamp/Engine-configure /build/source/build/src/engine/Engine-prefix/src/Engine-stamp/Engine-configure
       > cd /build/source/build/src/engine/skia && /build/source/src/engine/skia/bin/gn gen --root=/build/source/src/engine/skia /build/source/build/src/engine/skia "--args=ar=\"/nix/store/al7l1c2nwk9c9cicp3l7jfjxi1vilp93-clang-wrapper-21.1.2/bin/ar\" cc=\"/nix/store/al7l1c2nwk9c9cicp3l7jfjxi1vilp93-clang-wrapper-21.1.2/bin/clang\" cxx=\"/nix/store/al7l1c2nwk9c9cicp3l7jfjxi1vilp93-clang-wrapper-21.1.2/bin/clang++\" skia_use_egl=true is_component_build=true extra_cflags=[\"-Wno-error\", \"-Wno-psabi\"] is_official_build=true is_debug=false skia_enable_pdf=false skia_enable_skottie=false skia_enable_tools=false skia_use_dng_sdk=false skia_use_system_expat=true skia_use_system_libjpeg_turbo=true skia_use_system_libpng=true skia_use_system_libwebp=true skia_use_system_icu=true skia_use_system_harfbuzz=true skia_use_system_freetype2=true" && /nix/store/5bn5f4ivqf4xn19khh4kcg4ngnjs6spg-cmake-4.1.2/bin/cmake -E touch /build/source/build/src/engine/Engine-prefix/src/Engine-stamp/Engine-configure
       > [6/570] Building C object src/gperftools/CMakeFiles/logging.dir/src/base/dynamic_annotations.c.o
       > [7/570] Building CXX object src/gperftools/CMakeFiles/stacktrace_object.dir/src/base/vdso_support.cc.o
       > [8/570] Building CXX object src/gperftools/CMakeFiles/spinlock.dir/src/base/spinlock_internal.cc.o
       > [9/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/common.cc.o
       > [10/570] Building CXX object src/gperftools/CMakeFiles/maybe_threads.dir/src/maybe_threads.cc.o
       > [11/570] Building CXX object src/gperftools/CMakeFiles/spinlock.dir/src/base/spinlock.cc.o
       > [12/570] Building CXX object src/gperftools/CMakeFiles/spinlock.dir/src/base/atomicops-internals-x86.cc.o
       > [13/570] Building CXX object src/gperftools/CMakeFiles/logging.dir/src/base/logging.cc.o
       > [14/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/stack_trace_table.cc.o
       > [15/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/system-alloc.cc.o
       > [16/570] Building CXX object src/gperftools/CMakeFiles/stacktrace_object.dir/src/base/elf_mem_image.cc.o
       > [17/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/memfs_malloc.cc.o
       > [18/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/central_freelist.cc.o
       > [19/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/span.cc.o
       > [20/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/page_heap.cc.o
       > [21/570] Building CXX object src/gperftools/CMakeFiles/sysinfo.dir/src/base/sysinfo.cc.o
       > [22/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/internal_logging.cc.o
       > [23/570] Building CXX object src/gperftools/CMakeFiles/stacktrace_object.dir/src/stacktrace.cc.o
       > [24/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/sampler.cc.o
       > ninja: build stopped: subcommand failed.
       For full logs, run:
         nix log /nix/store/lrni0ajy0niyiaqvp7697mj4r4n6a67m-friction-1.0.0-rc.2.drv
```

I was surprised to find that what it was trying to run was an executable file that came with the repository.

```
[lyuk98@framework:~]$ nix shell nixpkgs#file
[lyuk98@framework:~]$ file /nix/store/mynji0gk0m39dim1f3q8a2xn0fbicg37-source/src/engine/skia/bin/gn
/nix/store/mynji0gk0m39dim1f3q8a2xn0fbicg37-source/src/engine/skia/bin/gn: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=2e4b282fb752d34738284ba66a12f3d89c680fe4, stripped
```

Even if it could locate the executable, I was certain that the process is going to fail. To solve the problem, I included `gn` as a dependency.

```nix
{
  # parameters
  gn,
}:
let
	# ffmpeg_4-headless, harfbuzz-icu, and qtVersion
in
clangStdenv.mkDerivation (finalAttrs: {
  # package metadata and source definition

  buildInputs = [
    # build inputs
  ];
  nativeBuildInputs = [
    # build inputs
    gn
  ];

  cmakeFlags = [
    # CMake flags
  ];
})
```

Even then, building still failed with the same error. After looking through the code, I found [the problematic part](https://github.com/friction2d/friction/blob/v1.0.0-rc.2/src/engine/CMakeLists.txt#L62 "friction/src/engine/CMakeLists.txt at v1.0.0-rc.2 · friction2d/friction"):

```cmake
    set(SKIA_ARGS "${SKIA_ARGS} extra_cflags=[\"-Wno-error\",\"/MD\",\"/O2\"]")
else()
    set(GN_PATH ${SKIA_SRC}/bin/gn)
    set(SKIA_BUILD_CMD ninja -j${N})
    if(${USE_SKIA_SYSTEM_LIBS})
```

With `SKIA_SRC` pointing to its submodule, the build process used its own executable, no matter the dependency. I had to modify the source and let the process use what was available in the environment; to do so, I created [a patch](https://github.com/lyuk98/nixos-config/blob/d11125d54ef2b30e2afb95bd7f68d101c16a35a3/packages/friction/gn-replace-executable.patch "nixos-config/packages/friction/gn-replace-executable.patch at d11125d54ef2b30e2afb95bd7f68d101c16a35a3 · lyuk98/nixos-config") that removes the hardcoded path.

```diff
--- a/src/engine/CMakeLists.txt
+++ b/src/engine/CMakeLists.txt
@@ -59,7 +59,7 @@ if(WIN32)
     set(SKIA_ARGS "${SKIA_ARGS} clang_win=\"C:\\Program Files\\LLVM\" cc=\"clang-cl\" cxx=\"clang-cl\"")
     set(SKIA_ARGS "${SKIA_ARGS} extra_cflags=[\"-Wno-error\",\"/MD\",\"/O2\"]")
 else()
-    set(GN_PATH ${SKIA_SRC}/bin/gn)
+    set(GN_PATH gn)
     set(SKIA_BUILD_CMD ninja -j${N})
     if(${USE_SKIA_SYSTEM_LIBS})
         set(SKIA_UPDATE_CMD true)
```

The derivation was then edited to use the patch:

```nix
{
  # parameters
}:
let
	# ffmpeg_4-headless, harfbuzz-icu, and qtVersion
in
clangStdenv.mkDerivation (finalAttrs: {
  # package metadata, source definition, and build inputs

  patches = [
    ./gn-replace-executable.patch
  ];

  # CMake flags
})
```

The build process still failed, but with a different error.

```
[lyuk98@framework:~/nixos-config]$ nix build .#friction
error: Cannot build '/nix/store/ysswa7s8c3ykyq8qizl16arbwx61i5jw-friction-1.0.0-rc.2.drv'.
       Reason: builder failed with exit code 1.
       Output paths:
         /nix/store/gmkpvvf9b2bmdxmbrzrclszfcsicbcyf-friction-1.0.0-rc.2
       Last 25 log lines:
       >   set_sources_assignment_filter([])
       >   ^----------------------------
       > [8/570] Building CXX object src/gperftools/CMakeFiles/stacktrace_object.dir/src/base/vdso_support.cc.o
       > FAILED: [code=1] src/engine/Engine-prefix/src/Engine-stamp/Engine-configure /build/source/build/src/engine/Engine-prefix/src/Engine-stamp/Engine-configure
       > cd /build/source/build/src/engine/skia && gn gen --root=/build/source/src/engine/skia /build/source/build/src/engine/skia "--args=ar=\"/nix/store/al7l1c2nwk9c9cicp3l7jfjxi1vilp93-clang-wrapper-21.1.2/bin/ar\" cc=\"/nix/store/al7l1c2nwk9c9cicp3l7jfjxi1vilp93-clang-wrapper-21.1.2/bin/clang\" cxx=\"/nix/store/al7l1c2nwk9c9cicp3l7jfjxi1vilp93-clang-wrapper-21.1.2/bin/clang++\" skia_use_egl=true is_component_build=true extra_cflags=[\"-Wno-error\", \"-Wno-psabi\"] is_official_build=true is_debug=false skia_enable_pdf=false skia_enable_skottie=false skia_enable_tools=false skia_use_dng_sdk=false skia_use_system_expat=true skia_use_system_libjpeg_turbo=true skia_use_system_libpng=true skia_use_system_libwebp=true skia_use_system_icu=true skia_use_system_harfbuzz=true skia_use_system_freetype2=true" && /nix/store/5bn5f4ivqf4xn19khh4kcg4ngnjs6spg-cmake-4.1.2/bin/cmake -E touch /build/source/build/src/engine/Engine-prefix/src/Engine-stamp/Engine-configure
       > [10/570] Building CXX object src/gperftools/CMakeFiles/logging.dir/src/base/logging.cc.o
       > [11/570] Building CXX object src/gperftools/CMakeFiles/spinlock.dir/src/base/spinlock.cc.o
       > [12/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/memfs_malloc.cc.o
       > [13/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/internal_logging.cc.o
       > [14/570] Building CXX object src/gperftools/CMakeFiles/spinlock.dir/src/base/atomicops-internals-x86.cc.o
       > [15/570] Building CXX object src/gperftools/CMakeFiles/maybe_threads.dir/src/maybe_threads.cc.o
       > [16/570] Building CXX object src/gperftools/CMakeFiles/stacktrace_object.dir/src/stacktrace.cc.o
       > [17/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/static_vars.cc.o
       > [18/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/symbolize.cc.o
       > [19/570] Building CXX object src/gperftools/CMakeFiles/stacktrace_object.dir/src/base/elf_mem_image.cc.o
       > [20/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/system-alloc.cc.o
       > [21/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/stack_trace_table.cc.o
       > [22/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/central_freelist.cc.o
       > [23/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/sampler.cc.o
       > [24/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/thread_cache.cc.o
       > [25/570] Building CXX object src/gperftools/CMakeFiles/sysinfo.dir/src/base/sysinfo.cc.o
       > [26/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/span.cc.o
       > [27/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/page_heap.cc.o
       > [28/570] Building CXX object src/gperftools/CMakeFiles/tcmalloc_internal_object.dir/src/malloc_hook.cc.o
       > ninja: build stopped: subcommand failed.
       For full logs, run:
         nix log /nix/store/ysswa7s8c3ykyq8qizl16arbwx61i5jw-friction-1.0.0-rc.2.drv
```

I later found out that the problematic function `set_sources_assignment_filter()` [was removed](https://chromium.googlesource.com/chromium/src/+/master/docs/no_sources_assignment_filter.md "Chromium Docs - No sources_assignment_filter") from recent versions of GN. It was also when I noticed that Friction has [their own fork](https://github.com/friction2d/gn "friction2d/gn: Build system for skia") of the meta-build system, although it was forked from a commit that was made in March 2020. I could not tell if this practice is okay, but I decided to use their code, anyway.

Being a relatively simpler derivation (compared to FFmpeg), I wrote one from scratch. I did not want to deal with [something about the last commit position](https://github.com/NixOS/nixpkgs/blob/2d68b3f4952a613af3fbaca458e919a3564a9544/pkgs/by-name/gn/gn/package.nix#L55-L64 "nixpkgs/pkgs/by-name/gn/gn/package.nix at 2d68b3f4952a613af3fbaca458e919a3564a9544 · NixOS/nixpkgs"), so I referred to [the commit that did "fix" the problem](https://github.com/friction2d/gn/commit/70a9617aad7c09642457b6296d35638b97375dad "Fix broken build · friction2d/gn@70a9617").

```nix
{
  # parameters except gn
  stdenv,
}:
let
  # ffmpeg_4-headless
  gn = stdenv.mkDerivation {
    pname = "gn";
    version = "0-unstable-2025-08-25";

    src = fetchFromGitHub {
      owner = "friction2d";
      repo = "gn";
      rev = "70a9617aad7c09642457b6296d35638b97375dad";
      hash = "sha256-R/HL/a47swDCYM4iwAvcOkJa7qZb8hvsj9ULitgUWuk=";
    };

    nativeBuildInputs = [
      ninja
      python3
    ];

    configurePhase = ''
      python build/gen.py
    '';
    buildPhase = ''
      ninja --verbose -C out -j $NIX_BUILD_CORES gn
    '';
    installPhase = ''
      install --verbose -D out/gn "$out/bin/gn"
    '';

    meta = {
      description = "Meta-build system that generates build files for Ninja";
      mainProgram = "gn";
      homepage = "https://gn.googlesource.com/gn";
      license = lib.licenses.bsd3;
      platforms = lib.platforms.unix;
    };
  };
  # harfbuzz-icu

  # qtVersion
in
clangStdenv.mkDerivation (finalAttrs: {
  # package details
})
```

## Removing FHS-reliant code

The build failed again, but the reason was not so obvious by reading the partial output.

```
[lyuk98@framework:~/nixos-config]$ nix build .#friction
error: Cannot build '/nix/store/d9k7c67hj9pr2vkhc94paibjddnd5f9z-friction-1.0.0-rc.2.drv'.
       Reason: builder failed with exit code 1.
       Output paths:
         /nix/store/rjii25pjkhx4vh5xij0dzp3g0iz9kidw-friction-1.0.0-rc.2
       Last 25 log lines:
       > 1 error generated.
       > [747/830] compile ../../../../src/engine/skia/src/pathops/SkDConicLineIntersection.cpp
       > [748/830] compile ../../../../src/engine/skia/src/pathops/SkDLineIntersection.cpp
       > [749/830] compile ../../../../src/engine/skia/src/pathops/SkDQuadLineIntersection.cpp
       > [750/830] compile ../../../../src/engine/skia/src/pathops/SkIntersections.cpp
       > [751/830] compile ../../../../src/engine/skia/src/pathops/SkOpCubicHull.cpp
       > [752/830] compile ../../../../src/engine/skia/src/pathops/SkOpContour.cpp
       > [753/830] compile ../../../../src/engine/skia/src/pathops/SkOpAngle.cpp
       > [754/830] compile ../../../../src/engine/skia/modules/particles/src/SkParticleBinding.cpp
       > [755/830] compile ../../../../src/engine/skia/modules/skshaper/src/SkShaper.cpp
       > [756/830] compile ../../../../src/engine/skia/src/pathops/SkOpBuilder.cpp
       > [757/830] compile ../../../../src/engine/skia/src/pathops/SkOpCoincidence.cpp
       > [758/830] compile ../../../../src/engine/skia/src/pathops/SkOpEdgeBuilder.cpp
       > [759/830] compile ../../../../src/engine/skia/src/pathops/SkOpSpan.cpp
       > [760/830] compile ../../../../src/engine/skia/modules/particles/src/SkParticleEffect.cpp
       > [761/830] compile ../../../../src/engine/skia/src/pathops/SkPathOpsCommon.cpp
       > [762/830] compile ../../../../src/engine/skia/src/sksl/SkSLIRGenerator.cpp
       > [763/830] compile ../../../../src/engine/skia/src/pathops/SkPathOpsConic.cpp
       > [764/830] compile ../../../../src/engine/skia/src/pathops/SkPathOpsAsWinding.cpp
       > [765/830] compile ../../../../src/engine/skia/src/pathops/SkOpSegment.cpp
       > ninja: build stopped: subcommand failed.
       > [132/570] Linking CXX static library src/gperftools/libtcmalloc_and_profiler.a
       > FAILED: [code=1] src/engine/Engine-prefix/src/Engine-stamp/Engine-build /build/source/build/src/engine/Engine-prefix/src/Engine-stamp/Engine-build
       > cd /build/source/build/src/engine/skia && ninja -j20 && /nix/store/5bn5f4ivqf4xn19khh4kcg4ngnjs6spg-cmake-4.1.2/bin/cmake -E touch /build/source/build/src/engine/Engine-prefix/src/Engine-stamp/Engine-build
       > ninja: build stopped: subcommand failed.
       For full logs, run:
         nix log /nix/store/d9k7c67hj9pr2vkhc94paibjddnd5f9z-friction-1.0.0-rc.2.drv
```

I read the full log to find the error, which stated that `hb.h` was missing.

```
[lyuk98@framework:~]$ nix log /nix/store/d9k7c67hj9pr2vkhc94paibjddnd5f9z-friction-1.0.0-rc.2.drv | grep --after-context=1 --before-context=1 error
-- Detecting CXX compile features - done
-- skia args: ar="/nix/store/al7l1c2nwk9c9cicp3l7jfjxi1vilp93-clang-wrapper-21.1.2/bin/ar" cc="/nix/store/al7l1c2nwk9c9cicp3l7jfjxi1vilp93-clang-wrapper-21.1.2/bin/clang" cxx="/nix/store/al7l1c2nwk9c9cicp3l7jfjxi1vilp93-clang-wrapper-21.1.2/bin/clang++" skia_use_egl=true is_component_build=true extra_cflags=["-Wno-error", "-Wno-psabi"] is_official_build=true is_debug=false skia_enable_pdf=false skia_enable_skottie=false skia_enable_tools=false skia_use_dng_sdk=false skia_use_system_expat=true skia_use_system_libjpeg_turbo=true skia_use_system_libpng=true skia_use_system_libwebp=true skia_use_system_icu=true skia_use_system_harfbuzz=true skia_use_system_freetype2=true
-- Performing Test i386
--
FAILED: [code=1] obj/modules/skshaper/src/libskshaper.SkShaper_harfbuzz.o 
/nix/store/al7l1c2nwk9c9cicp3l7jfjxi1vilp93-clang-wrapper-21.1.2/bin/clang++ -MD -MF obj/modules/skshaper/src/libskshaper.SkShaper_harfbuzz.o.d -DSKSHAPER_IMPLEMENTATION=1 -DNDEBUG -DSK_GAMMA_APPLY_TO_A8 -DSKSHAPER_DLL -DSK_SHAPER_HARFBUZZ_AVAILABLE -DSK_GL -DSK_CODEC_DECODES_JPEG -DSK_ENCODE_JPEG -DSK_CODEC_DECODES_PNG -DSK_ENCODE_PNG -DSK_CODEC_DECODES_WEBP -DSK_ENCODE_WEBP -DSK_XML -DSKIA_DLL -DSK_R32_SHIFT=16 -DU_USING_ICU_NAMESPACE=0 -I../../../../src/engine/skia/modules/skshaper/include -I../../../../src/engine/skia -I/usr/include/harfbuzz -Wno-attributes -fstrict-aliasing -fPIC -fvisibility=hidden -O3 -fdata-sections -ffunction-sections -Wno-unused-parameter -Wno-error -Wno-psabi -std=c++17 -fvisibility-inlines-hidden -fno-exceptions -fno-rtti -c ../../../../src/engine/skia/modules/skshaper/src/SkShaper_harfbuzz.cpp -o obj/modules/skshaper/src/libskshaper.SkShaper_harfbuzz.o
../../../../src/engine/skia/modules/skshaper/src/SkShaper_harfbuzz.cpp:32:10: fatal error: 'hb.h' file not found
   32 | #include <hb.h>
      |          ^~~~~~
1 error generated.
[747/830] compile ../../../../src/engine/skia/src/pathops/SkDConicLineIntersection.cpp
```

It confused me, since [HarfBuzz](https://harfbuzz.github.io/ "HarfBuzz Manual: HarfBuzz Manual") is already included as one of the `buildInputs`. After digging through the source code, however, I found [the problematic code](https://github.com/friction2d/skia/blob/4e897d0bec0ee279f77320db76f89e47eb60b7e4/third_party/harfbuzz/BUILD.gn#L15) from Friction's fork of [Skia](https://skia.org/ "Skia"):

```gn
  system("harfbuzz") {
    include_dirs = [ "/usr/include/harfbuzz" ]
    libs = [ "harfbuzz" ]
  }
```

NixOS is not compliant with [Filesystem Hierarchy Standard (FHS)](https://refspecs.linuxfoundation.org/fhs.shtml "FHS Referenced Specifications") (with packages being stored within `/nix/store`), so I had to replace the directory with where the header will actually be. Since it would be tedious to change the store path within static patch files, a `postPatch` script was added to dynamically replace `/usr`.

```nix
{
  # parameters
}:
let
  # ffmpeg_4-headless, gn, harfbuzz-icu, and qtVersion
in
clangStdenv.mkDerivation (finalAttrs: {
  # package metadata, source definition, and static patches

  postPatch = ''
    substituteInPlace src/engine/skia/third_party/freetype2/BUILD.gn \
      --replace-fail /usr ${freetype.dev}
    substituteInPlace src/engine/skia/third_party/harfbuzz/BUILD.gn \
      --replace-fail /usr ${harfbuzz-icu.dev}
  '';

  # build inputs and CMake flags
})
```

And yes, the same problem occurred with [FreeType](https://freetype.org/ "The FreeType Project"), which I subsequently solved in the same way.

## Adding some details

The build finally succeeded. I ran the application to appreciate the effort I have made, but soon noticed a small difference to the AppImage version:

| AppImage | Nix package |
| --- | --- |
| ![About window from Friction. Under its logo and title, there are lines of text: "1.0.0-rc.2", "NixOS 25.11 (Xantusia) x86_64 wayland", "Mesa Intel(R) Xe Graphics (ADL GT2)", "Qt 5.15.17", and "Built from commit e1e2c9a2"](https://images.lyuk98.com/4a907fe6-95f5-45d7-9082-2079fa74034c.avif "The AppImage version") | <picture><source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/e317d139-1206-4309-9f51-df9800093973.avif"><img src="https://images.lyuk98.com/7212ba92-21b0-45bc-abc3-0aa4ed9323e4.avif" alt="About window from Friction. Under its logo and title, there are lines of text: &quot;1.0.0-dev&quot;, &quot;NixOS 25.11 (Xantusia) x86_64 X11&quot;, &quot;Mesa Intel(R) Xe Graphics (ADL GT2)&quot;, and &quot;Qt 5.15.17&quot;" title="The Nix package version"></picture> |

To change them, I modified the CMake flags a bit. To get the commit ID, I also changed `src`, fetching source based on `rev` instead of `tag`.

```nix
{
  # parameters
}:
let
  # ffmpeg_4-headless, gn, harfbuzz-icu, and qtVersion
in
clangStdenv.mkDerivation (finalAttrs: {
  # package metadata

  src = fetchFromGitHub {
    owner = "friction2d";
    repo = "friction";
    rev = "e1e2c9a20a60bbaa9c0d618b88e37ce35185cd7c";
    fetchSubmodules = true;
    hash = "sha256-tmxzEfOy+eCe7K4Rv+bFNk0t3aD1n4iqAroB1li9vVM=";
  };

  # patches and build inputs

  cmakeFlags = [
    "-DCMAKE_BUILD_TYPE=Release"

    "-DFRICTION_OFFICIAL_RELEASE=ON"
    "-DCUSTOM_BUILD=rc.2"

    "-DGIT_COMMIT=${builtins.substring 0 8 finalAttrs.src.rev}"
    "-DGIT_BRANCH=v${lib.versions.majorMinor finalAttrs.version}"

    "-DQSCINTILLA_INCLUDE_DIRS=${qscintilla}/include"
    "-DQSCINTILLA_LIBRARIES_DIRS=${qscintilla}/lib"
    "-DQSCINTILLA_LIBRARIES=qscintilla2_qt${qtVersion}"
  ];
})
```

## The result

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/a9ae615a-ed57-468a-8ec8-da5537a05e40.avif">
  <img src="https://images.lyuk98.com/ec96a2d9-f0bf-4bae-aa94-b5d9370d3199.avif" alt="The main window of Friction" title="The result">
</picture>

The application was finally running. Some features may be broken since I have not tested anything yet, but fixing them will be up to my future self.

# Building Open Vehicle Diagnostics

[![Logo of Open Vehicle Diagnostics](https://images.lyuk98.com/260ddc64-f912-4de8-938f-716cac70a68e.png)](https://github.com/rnd-ash/OpenVehicleDiag "rnd-ash/OpenVehicleDiag: A rust based cross-platform ECU diagnostics and car hacking application, utilizing the passthru protocol")

> Open Vehicle Diagnostics (OVD) is a Rust-based open source vehicle ECU diagnostic platform that makes use of the J2534-2 protocol, as well as SocketCAN on Linux!

## Creating a derivation

It has not been actively maintained, but I wanted to try and see what I can do with it. First, the package was defined at `packages/default.nix`.

```nix
{
  pkgs ? import <nixpkgs> { },
  ...
}:
{
  # personal programs

  # Packages from the internet
  friction = pkgs.libsForQt5.callPackage ./friction { };
  openvehiclediag = pkgs.callPackage ./openvehiclediag { };
}
```

I have tried writing a Rust derivation before, so I sort of knew where to start. A new one was created at `packages/openvehiclediag/default.nix`.

```nix
{
  fetchFromGitHub,
  lib,
  rustPlatform,
}:
rustPlatform.buildRustPackage (finalAttrs: {
  pname = "openvehiclediag";
  version = "1.0.5";

  meta = {
    description = "Cross-platform ECU diagnostics and car hacking application";
    longDescription = ''
      Open Vehicle Diagnostics (OVD) is a Rust-based open source vehicle ECU diagnostic platform
      that makes use of the J2534-2 protocol, as well as SocketCAN on Linux!
    '';
    homepage = "https://github.com/rnd-ash/OpenVehicleDiag";
    changelog = "https://github.com/rnd-ash/OpenVehicleDiag/releases/tag/v${finalAttrs.version}";
    license = lib.licenses.gpl3;
    mainProgram = "openvehiclediag";
  };

  src = fetchFromGitHub {
    owner = "rnd-ash";
    repo = "OpenVehicleDiag";
    tag = "v${finalAttrs.version}";
    hash = lib.fakeHash;
    rootDir = "app_rust";
  };

  cargoHash = lib.fakeHash;
})
```

## The Cargo problem

The next step was to intentionally fail building to get actual hash values. I could get one for the source code:

```
[lyuk98@framework:~/nixos-config]$ nix build .#openvehiclediag
error: hash mismatch in fixed-output derivation '/nix/store/0k1b1y21g6mgl6r6nzk0id072k394lzq-source.drv':
         specified: sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=
            got:    sha256-LR9fr9n5Ag4jqO0mY6bXPCi8YurqzIfPh7s8nInnFz0=
error: Cannot build '/nix/store/lgsrgw86nm48i9691929c2mrj44q14r8-openvehiclediag-1.0.5.drv'.
       Reason: 1 dependency failed.
       Output paths:
         /nix/store/7lh8h5vrnn42jrwlgg70cdpxv4phqk62-openvehiclediag-1.0.5
```

...but an unexpected error showed up afterwards, before even complaining about hash mismatch:

```
[lyuk98@framework:~/nixos-config]$ nix build .#openvehiclediag
error: Cannot build '/nix/store/zd191nhzvr241xbfsx2n7p84dpia8rbf-openvehiclediag-1.0.5-vendor-staging.drv'.
       Reason: builder failed with exit code 1.
       Output paths:
         /nix/store/2rpgc2cdw2jd5h7drs2gjkz2cvw3lhky-openvehiclediag-1.0.5-vendor-staging
       Last 22 log lines:
       > Running phase: unpackPhase
       > unpacking source archive /nix/store/75b59hlrp99igl6islx68k5jw258pjsp-source
       > source root is source
       > Running phase: patchPhase
       > Running phase: updateAutotoolsGnuConfigScriptsPhase
       > Running phase: buildPhase
       > Traceback (most recent call last):
       >   File "/nix/store/7456ivaaxn7fwz98sx8vqsx1zfnz53f4-fetch-cargo-vendor-util/bin/fetch-cargo-vendor-util", line 349, in <module>
       >     main()
       >     ~~~~^^
       >   File "/nix/store/7456ivaaxn7fwz98sx8vqsx1zfnz53f4-fetch-cargo-vendor-util/bin/fetch-cargo-vendor-util", line 345, in main
       >     subcommand_func()
       >     ~~~~~~~~~~~~~~~^^
       >   File "/nix/store/7456ivaaxn7fwz98sx8vqsx1zfnz53f4-fetch-cargo-vendor-util/bin/fetch-cargo-vendor-util", line 336, in <lambda>
       >     "create-vendor-staging": lambda: create_vendor_staging(lockfile_path=Path(sys.argv[2]), out_dir=Path(sys.argv[3])),
       >                                      ~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
       >   File "/nix/store/7456ivaaxn7fwz98sx8vqsx1zfnz53f4-fetch-cargo-vendor-util/bin/fetch-cargo-vendor-util", line 127, in create_vendor_staging
       >     cargo_lock_toml = load_toml(lockfile_path)
       >   File "/nix/store/7456ivaaxn7fwz98sx8vqsx1zfnz53f4-fetch-cargo-vendor-util/bin/fetch-cargo-vendor-util", line 23, in load_toml
       >     with open(path, "rb") as f:
       >          ~~~~^^^^^^^^^^^^
       > FileNotFoundError: [Errno 2] No such file or directory: 'Cargo.lock'
       For full logs, run:
         nix log /nix/store/zd191nhzvr241xbfsx2n7p84dpia8rbf-openvehiclediag-1.0.5-vendor-staging.drv
error: Cannot build '/nix/store/kfi0fzpacqkk868bq4ayyvxir8r5992p-openvehiclediag-1.0.5-vendor.drv'.
       Reason: 1 dependency failed.
       Output paths:
         /nix/store/ncmb3ppxsa9c4kr069g77d5lpqx48zfv-openvehiclediag-1.0.5-vendor
error: Cannot build '/nix/store/00yi4w8lii2815c5s967xl7pl22pxva0-openvehiclediag-1.0.5.drv'.
       Reason: 1 dependency failed.
       Output paths:
         /nix/store/4gp0rsvxzx3w77788zzziq6w2r0cv9bh-openvehiclediag-1.0.5
```

Right, `Cargo.lock` was missing. I generated one myself and copied it to my repository.

```
[lyuk98@framework:~]$ nix shell nixpkgs#cargo
[lyuk98@framework:~]$ git clone https://github.com/rnd-ash/OpenVehicleDiag.git
[lyuk98@framework:~]$ cd OpenVehicleDiag/app_rust/
[lyuk98@framework:~/OpenVehicleDiag/app_rust]$ git switch --detach v1.0.5
[lyuk98@framework:~/OpenVehicleDiag/app_rust]$ cargo generate-lockfile
[lyuk98@framework:~/OpenVehicleDiag/app_rust]$ cp Cargo.lock ~/nixos-config/packages/openvehiclediag/Cargo.lock
```

The lockfile was then referred by the derivation. Notably, `cargoHash` was also omitted.

```nix
{
  fetchFromGitHub,
  lib,
  rustPlatform,
}:
rustPlatform.buildRustPackage (finalAttrs: {
  # package metadata

  src = fetchFromGitHub {
    owner = "rnd-ash";
    repo = "OpenVehicleDiag";
    tag = "v${finalAttrs.version}";
    hash = "sha256-LR9fr9n5Ag4jqO0mY6bXPCi8YurqzIfPh7s8nInnFz0=";
    rootDir = "app_rust";
  };

  cargoLock = {
    lockFile = ./Cargo.lock;
  };
})
```

However, another build failure greeted me afterwards.

```
[lyuk98@framework:~/nixos-config]$ nix build .#openvehiclediag
error:
       … while calling the 'derivationStrict' builtin
         at <nix/derivation-internal.nix>:37:12:
           36|
           37|   strict = derivationStrict drvAttrs;
             |            ^
           38|

       … while evaluating derivation 'openvehiclediag-1.0.5'
         whose name attribute is located at /nix/store/sqilaxdxljyalv530z1qrd1q3nd4lyxq-source/pkgs/stdenv/generic/make-derivation.nix:544:13

       … while evaluating attribute 'cargoDeps' of derivation 'openvehiclediag-1.0.5'
         at /nix/store/sqilaxdxljyalv530z1qrd1q3nd4lyxq-source/pkgs/build-support/rust/build-rust-package/default.nix:85:7:
           84|     // {
           85|       cargoDeps =
             |       ^
           86|         if cargoVendorDir != null then

       (stack trace truncated; use '--show-trace' to show the full, detailed trace)

       error: No hash was found while vendoring the git dependency j2534_rust-1.5.0. You can add
       a hash through the `outputHashes` argument of `importCargoLock`:

       outputHashes = {
         "j2534_rust-1.5.0" = "<hash>";
       };

       If you use `buildRustPackage`, you can add this attribute to the `cargoLock`
       attribute set.
```

It seemed like `Cargo.lock` does not pin the exact version (or commit ID in this case) of its dependencies from Git. To mitigate this, I added one of the `outputHashes` for Cargo dependencies:

```nix
{
  fetchFromGitHub,
  lib,
  rustPlatform,
}:
rustPlatform.buildRustPackage (finalAttrs: {
  # package metadata and source declaration

  cargoLock = {
    lockFile = ./Cargo.lock;

    outputHashes = {
      "j2534_rust-1.5.0" = "sha256-NlQ9z9R3jrAz5xDtwxq4NtE17+oJXIVF11r2C+jNAPI=";
    };
  };
})
```

The build still failed. A `Cargo.lock` was missing somewhere in this code.

```
[lyuk98@framework:~/nixos-config]$ nix build .#openvehiclediag
error: Cannot build '/nix/store/ra5rx8zb9zj4fchjg93g0g1kkg67kyh5-openvehiclediag-1.0.5.drv'.
       Reason: builder failed with exit code 1.
       Output paths:
         /nix/store/w9ri5yx81pc0slcl2520kbw6zc99w6gm-openvehiclediag-1.0.5
       Last 11 log lines:
       > Running phase: unpackPhase
       > unpacking source archive /nix/store/75b59hlrp99igl6islx68k5jw258pjsp-source
       > source root is source
       > Executing cargoSetupPostUnpackHook
       > Finished cargoSetupPostUnpackHook
       > Running phase: patchPhase
       > Executing cargoSetupPostPatchHook
       > Validating consistency between /build/source/Cargo.lock and /build/cargo-vendor-dir/Cargo.lock
       > /nix/store/7ql4x9i7w7ihxw23vkanvcvrvqhay23c-diffutils-3.12/bin/diff: /build/source/Cargo.lock: No such file or directory
       > ERROR: Missing Cargo.lock from src. Expected to find it at: /build/source/Cargo.lock
       > Hint: You can use the cargoPatches attribute to add a Cargo.lock manually to the build.
       For full logs, run:
         nix log /nix/store/ra5rx8zb9zj4fchjg93g0g1kkg67kyh5-openvehiclediag-1.0.5.drv
```

It was at that time when I noticed that this repository consists of several parts. I decided to build both [the GUI app](https://github.com/rnd-ash/OpenVehicleDiag/tree/0d9a6bdf88e219a6b9375b45a638cc5acc2fbcf3/app_rust "OpenVehicleDiag/app_rust at 0d9a6bdf88e219a6b9375b45a638cc5acc2fbcf3 · rnd-ash/OpenVehicleDiag") and [CBFParser](https://github.com/rnd-ash/OpenVehicleDiag/tree/0d9a6bdf88e219a6b9375b45a638cc5acc2fbcf3/CBFParser "OpenVehicleDiag/CBFParser at 0d9a6bdf88e219a6b9375b45a638cc5acc2fbcf3 · rnd-ash/OpenVehicleDiag"), and created `Cargo.lock` for all necessary parts.

```
[lyuk98@framework:~]$ nix shell nixpkgs#cargo
[lyuk98@framework:~]$ git clone https://github.com/rnd-ash/OpenVehicleDiag.git
[lyuk98@framework:~]$ cd OpenVehicleDiag/
[lyuk98@framework:~/OpenVehicleDiag]$ git switch --detach v1.0.5
[lyuk98@framework:~/OpenVehicleDiag]$ cargo generate-lockfile --manifest-path CBFParser/Cargo.toml
[lyuk98@framework:~/OpenVehicleDiag]$ cargo generate-lockfile --manifest-path common/Cargo.toml
[lyuk98@framework:~/OpenVehicleDiag]$ cp CBFParser/Cargo.lock ~/nixos-config/packages/openvehiclediag/cbf_parser.Cargo.lock
[lyuk98@framework:~/OpenVehicleDiag]$ cp common/Cargo.lock ~/nixos-config/packages/openvehiclediag/common.Cargo.lock
```

While I was at it, I decided to be specific about the GUI application's dependency (from a Git repository), too. Instead of the ambiguous `main` branch, I specified the latest commit ID at the time of writing, which I later separately applied within a patch script.

```
[lyuk98@framework:~/OpenVehicleDiag]$ sed --in-place 's/branch="main"/rev="d8008b0c01809cb10e777d5ab64d5111ab981bb2"/g' app_rust/Cargo.toml
[lyuk98@framework:~/OpenVehicleDiag]$ cargo generate-lockfile --manifest-path app_rust/Cargo.toml
[lyuk98@framework:~/OpenVehicleDiag]$ cp app_rust/Cargo.lock ~/nixos-config/packages/openvehiclediag/openvehiclediag.Cargo.lock
```

Previous `Cargo.lock` was removed, and the following is what I ended up with:

```nix
{
  fetchFromGitHub,
  lib,
  rustPlatform,

  program ? "openvehiclediag",
}:
let
  rootDir = {
    "openvehiclediag" = "app_rust";
    "cbf_parser" = "CBFParser";
    "common" = "common";
  };
  version = {
    "openvehiclediag" = "1.0.5";
    "cbf_parser" = "0.1.0";
    "common" = "0.1.0";
  };
  lockfile = {
    "openvehiclediag" = ./openvehiclediag.Cargo.lock;
    "cbf_parser" = ./cbf_parser.Cargo.lock;
    "common" = ./common.Cargo.lock;
  };
in
assert builtins.elem program [
  "openvehiclediag"
  "cbf_parser"
];
rustPlatform.buildRustPackage (finalAttrs: {
  pname = program;
  version = version.${program};

  meta =
    let
      description = {
        openvehiclediag = "Cross-platform ECU diagnostics and car hacking application";
        cbf_parser = "Parses Mercedes CBF Files into OpenVehicleDiag's JSON";
      };
      longDescription = {
        openvehiclediag = ''
          Open Vehicle Diagnostics (OVD) is a Rust-based open source vehicle ECU diagnostic
          platform that makes use of the J2534-2 protocol, as well as SocketCAN on Linux!
        '';
        cbf_parser = ''
          This program converts Daimler CBF Files to the OVD JSON Schema. It can also be used to
          translate all the German strings in the CBF to a language of your choosing!
        '';
      };
    in
    {
      description = description.${program};
      longDescription = longDescription.${program};
      homepage = "https://github.com/rnd-ash/OpenVehicleDiag";
      changelog = "https://github.com/rnd-ash/OpenVehicleDiag/releases/tag/v${version.openvehiclediag}";
      license = lib.licenses.gpl3;
      mainProgram = program;
    };

  src = fetchFromGitHub {
    owner = "rnd-ash";
    repo = "OpenVehicleDiag";
    tag = "v${version.openvehiclediag}";
    hash = "sha256-RXVCEXR5rwygBySPHbsvYjBkMkHFVhRbcqUFvefpvaU=";
  };

  postPatch = ''
    substituteInPlace ${rootDir.openvehiclediag}/Cargo.toml \
      --replace-fail 'branch="main"' 'rev="d8008b0c01809cb10e777d5ab64d5111ab981bb2"'

    ln --symbolic --verbose ${lockfile.openvehiclediag} ${rootDir.openvehiclediag}/Cargo.lock
    ln --symbolic --verbose ${lockfile.cbf_parser} ${rootDir.cbf_parser}/Cargo.lock
    ln --symbolic --verbose ${lockfile.common} ${rootDir.common}/Cargo.lock
  '';

  cargoRoot = rootDir.${program};
  cargoLock.lockFile = lockfile.${program};
  buildAndTestSubdir = finalAttrs.cargoRoot;

  cargoDeps =
    if (program == "openvehiclediag") then
      (rustPlatform.fetchCargoVendor {
        src = finalAttrs.src;
        postPatch = finalAttrs.postPatch;
        cargoRoot = finalAttrs.cargoRoot;

        hash = "sha256-gNJjueF+G+sjj4TY7ITW2mcdrrV5/B//Gj1ibD9IbBM=";
      })
    else
      null;
})
```

Before trying to build the package, I also modified `packages/default.nix` so that CBFParser can be used separately without manually overriding `openvehiclediag`.

```nix
{
  pkgs ? import <nixpkgs> { },
  ...
}:
{
  # personal programs

  # Packages from the internet
  friction = pkgs.libsForQt5.callPackage ./friction { };
  openvehiclediag = pkgs.callPackage ./openvehiclediag { };
  cbf-parser = pkgs.callPackage ./openvehiclediag { program = "cbf_parser"; };
}
```

With everything done so far, I could run CBFParser:

```
[lyuk98@framework:~/nixos-config]$ nix run .#cbf-parser
Error: Invalid number of args: 0
Usage:
cbf_parser <INPUT.CBF>
cbf_parser <INPUT.CBF> -dump_strings <STRINGS.csv>
cbf_parser <INPUT.CBF> -load_strings <STRINGS.csv>
```

...but not the GUI application, which complained that CMake is missing:

```
[lyuk98@framework:~/nixos-config]$ nix build .#openvehiclediag
error: Cannot build '/nix/store/k8bgh9jp0y0ni2v8v95xj0kp9b97q80d-openvehiclediag-1.0.5.drv'.
       Reason: builder failed with exit code 101.
       Output paths:
         /nix/store/px1glgsmi4crmcbcpylpjh78lcv8vg5h-openvehiclediag-1.0.5
       Last 25 log lines:
       >   CMAKE_TOOLCHAIN_FILE = None
       >   CMAKE_GENERATOR_x86_64-unknown-linux-gnu = None
       >   CMAKE_GENERATOR_x86_64_unknown_linux_gnu = None
       >   HOST_CMAKE_GENERATOR = None
       >   CMAKE_GENERATOR = None
       >   CMAKE_PREFIX_PATH_x86_64-unknown-linux-gnu = None
       >   CMAKE_PREFIX_PATH_x86_64_unknown_linux_gnu = None
       >   HOST_CMAKE_PREFIX_PATH = None
       >   CMAKE_PREFIX_PATH = None
       >   CMAKE_x86_64-unknown-linux-gnu = None
       >   CMAKE_x86_64_unknown_linux_gnu = None
       >   HOST_CMAKE = None
       >   CMAKE = None
       >   running: cd "/build/source/target/x86_64-unknown-linux-gnu/release/build/freetype-sys-4ec77d7490b8458f/out/build" && CMAKE_PREFIX_PATH="" LC_ALL="C" "cmake" "/build/cargo-deps-vendor/freetype-sys-0.13.1/freetype2" "-DWITH_BZip2=OFF" "-DWITH_HarfBuzz=OFF" "-DWITH_PNG=OFF" "-DWITH_ZLIB=OFF" "-DCMAKE_INSTALL_PREFIX=/build/source/target/x86_64-unknown-linux-gnu/release/build/freetype-sys-4ec77d7490b8458f/out" "-DCMAKE_C_FLAGS= -ffunction-sections -fdata-sections -fPIC -m64 -fno-omit-frame-pointer" "-DCMAKE_C_COMPILER=/nix/store/x8mydcgbry214s802nzvy7fdljx404ym-gcc-wrapper-14.3.0/bin/cc" "-DCMAKE_CXX_FLAGS= -ffunction-sections -fdata-sections -fPIC -m64 -fno-omit-frame-pointer" "-DCMAKE_CXX_COMPILER=/nix/store/x8mydcgbry214s802nzvy7fdljx404ym-gcc-wrapper-14.3.0/bin/c++" "-DCMAKE_ASM_FLAGS= -ffunction-sections -fdata-sections -fPIC -m64 -fno-omit-frame-pointer" "-DCMAKE_ASM_COMPILER=/nix/store/x8mydcgbry214s802nzvy7fdljx404ym-gcc-wrapper-14.3.0/bin/cc" "-DCMAKE_BUILD_TYPE=Release"
       >
       >   --- stderr
       >
       >   thread 'main' panicked at /build/cargo-deps-vendor/cmake-0.1.54/src/lib.rs:1119:5:
       >
       >   failed to execute command: No such file or directory (os error 2)
       >   is `cmake` not installed?
       >
       >   build script failed, must exit now
       >   note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
       > warning: build failed, waiting for other jobs to finish...
       For full logs, run:
         nix log /nix/store/k8bgh9jp0y0ni2v8v95xj0kp9b97q80d-openvehiclediag-1.0.5.drv
```

I was initially confused, because no part of the code ever mentioned the build tool. However, after thinking about it for a while, it was apparent that some of the application's dependencies are [Foreign Function Interface (FFI)](https://doc.rust-lang.org/nomicon/ffi.html "FFI - The Rustonomicon") bindings, thus requiring their corresponding native libraries.

```nix
{
  fetchFromGitHub,
  lib,
  rustPlatform,

  gtk3,
  pkg-config,

  program ? "openvehiclediag",
}:
let
  # rootDir, version, and lockfile
in
assert builtins.elem program [
  "openvehiclediag"
  "cbf_parser"
];
rustPlatform.buildRustPackage (finalAttrs: {
  # package metadata and source information

  buildInputs = lib.optionals (program == "openvehiclediag") [
    gtk3
  ];
  nativeBuildInputs = lib.optionals (program == "openvehiclediag") [
    pkg-config
  ];

  # Cargo properties
})
```

## Modifying the source

It looked like the problems regarding dependencies were fixed at this point. However, another build problem that occurred afterwards was due to errors from the diagnostic software's code.

```
[lyuk98@framework:~/nixos-config]$ nix build .#openvehiclediag
error: Cannot build '/nix/store/s8470zzkpdb7g0lb6zd3f4c4znghz192-openvehiclediag-1.0.5.drv'.
       Reason: builder failed with exit code 101.
       Output paths:
         /nix/store/z8wdgg2ilw9992gp0p7r33s2nahjlgjp-openvehiclediag-1.0.5
       Last 25 log lines:
       > 28 |     pub fn new(comm_server: Box<dyn ComServer>, ecu: ISO15765Config) -> SessionResult<Self> {
       >    |                ^^^^^^^^^^^ help: if this is intentional, prefix it with an underscore: `_comm_server`
       >
       > warning: unused variable: `ecu`
       >   --> src/windows/diag_session/custom_session.rs:28:49
       >    |
       > 28 |     pub fn new(comm_server: Box<dyn ComServer>, ecu: ISO15765Config) -> SessionResult<Self> {
       >    |                                                 ^^^ help: if this is intentional, prefix it with an underscore: `_ecu`
       >
       > warning: unused variable: `msg`
       >   --> src/windows/diag_session/custom_session.rs:46:26
       >    |
       > 46 |     fn update(&mut self, msg: &Self::msg) -> Option<Self::msg> {
       >    |                          ^^^ help: if this is intentional, prefix it with an underscore: `_msg`
       >
       > warning: unused variable: `write_res`
       >   --> src/main.rs:48:13
       >    |
       > 48 |         let write_res = File::create(&path).unwrap().write_all(report.as_bytes());
       >    |             ^^^^^^^^^ help: if this is intentional, prefix it with an underscore: `_write_res`
       >
       > Some errors have detailed explanations: E0432, E0599, E0605.
       > For more information about an error, try `rustc --explain E0432`.
       > warning: `openvehiclediag` (bin "openvehiclediag") generated 43 warnings
       > error: could not compile `openvehiclediag` (bin "openvehiclediag") due to 6 previous errors; 57 warnings emitted
       For full logs, run:
         nix log /nix/store/s8470zzkpdb7g0lb6zd3f4c4znghz192-openvehiclediag-1.0.5.drv
```

I read the full log and found the following errors:

```
error[E0432]: unresolved import `j2534_rust::Loggable`
 --> src/commapi/passthru_api.rs:9:40
  |
9 |     ConnectFlags, IoctlID, IoctlParam, Loggable, PassthruError, Protocol, SConfig, SConfigList,
  |                                        ^^^^^^^^ no `Loggable` in the root
```

```
error[E0605]: non-primitive cast: `ConnectFlags` as `u32`
   --> src/commapi/passthru_api.rs:163:22
    |
163 |             flags |= ConnectFlags::CAN_29BIT_ID as u32;
    |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ an `as` expression can only be used to convert between primitive types or to coerce to a specific trait object

error[E0605]: non-primitive cast: `ConnectFlags` as `u32`
   --> src/commapi/passthru_api.rs:208:22
    |
208 |             flags |= ConnectFlags::CAN_29BIT_ID as u32;
    |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ an `as` expression can only be used to convert between primitive types or to coerce to a specific trait object

error[E0605]: non-primitive cast: `ConnectFlags` as `u32`
   --> src/commapi/passthru_api.rs:211:22
    |
211 |             flags |= ConnectFlags::ISO15765_ADDR_TYPE as u32;
    |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ an `as` expression can only be used to convert between primitive types or to coerce to a specific trait object
```

```
error[E0599]: no variant or associated item named `from_raw` found for enum `j2534_rust::PassthruError` in the current scope
   --> src/passthru.rs:138:33
    |
138 |         _ => Err(PassthruError::from_raw(res as u32).unwrap()),
    |                                 ^^^^^^^^ variant or associated item not found in `j2534_rust::PassthruError`
    |
help: there is an associated function `from` with a similar name
    |
138 -         _ => Err(PassthruError::from_raw(res as u32).unwrap()),
138 +         _ => Err(PassthruError::from(res as u32).unwrap()),
    |

error[E0599]: no variant or associated item named `from_raw` found for enum `j2534_rust::PassthruError` in the current scope
   --> src/passthru.rs:440:37
    |
440 |             _ => Err(PassthruError::from_raw(res as u32).unwrap()),
    |                                     ^^^^^^^^ variant or associated item not found in `j2534_rust::PassthruError`
    |
help: there is an associated function `from` with a similar name
    |
440 -             _ => Err(PassthruError::from_raw(res as u32).unwrap()),
440 +             _ => Err(PassthruError::from(res as u32).unwrap()),
    |
```

The problem largely looked like a result of breaking changes made to the author's [J2534 protocol implementation](https://github.com/rnd-ash/J2534-Rust "rnd-ash/J2534-Rust: Cross platform J2534 definition for Rust"). The latest commit at the time of writing [removed](https://github.com/rnd-ash/J2534-Rust/commit/d8008b0c01809cb10e777d5ab64d5111ab981bb2#diff-b1a35a68f14e696205874893c07fd24fdb88882b47c23cc0e0c80a30c7d53759L13-L15 "Update codebase to remove unneeded deps · rnd-ash/J2534-Rust@d8008b0") the `Loggable` trait, therefore causing one of the errors I have just encountered.

I decided to change the commit ID of the dependency, but before then, I tried building Open Vehicle Diagnostics with the newest commit I could get. With [some interesting changes](https://github.com/rnd-ash/OpenVehicleDiag/compare/v1.0.5...0d9a6bdf88e219a6b9375b45a638cc5acc2fbcf3 "Comparing v1.0.5...0d9a6bdf88e219a6b9375b45a638cc5acc2fbcf3 · rnd-ash/OpenVehicleDiag"), I thought there could be improvements. It was also not like I am submitting this package to either Nixpkgs or NUR (for now), anyway.

While doing so, I had to recreate `Cargo.lock` files.

```
[lyuk98@framework:~/OpenVehicleDiag]$ git restore .
[lyuk98@framework:~/OpenVehicleDiag]$ git switch --detach 0d9a6bdf88e219a6b9375b45a638cc5acc2fbcf3
[lyuk98@framework:~/OpenVehicleDiag]$ sed --in-place 's/branch="main"/rev="d8008b0c01809cb10e777d5ab64d5111ab981bb2"/g' app_rust/Cargo.toml
[lyuk98@framework:~/OpenVehicleDiag]$ cargo generate-lockfile --manifest-path app_rust/Cargo.toml
[lyuk98@framework:~/OpenVehicleDiag]$ cargo generate-lockfile --manifest-path CBFParser/Cargo.toml
[lyuk98@framework:~/OpenVehicleDiag]$ cargo generate-lockfile --manifest-path common/Cargo.toml
[lyuk98@framework:~/OpenVehicleDiag]$ cp app_rust/Cargo.lock ~/nixos-config/packages/openvehiclediag/openvehiclediag.Cargo.lock
[lyuk98@framework:~/OpenVehicleDiag]$ cp CBFParser/Cargo.lock ~/nixos-config/packages/openvehiclediag/cbf_parser.Cargo.lock
[lyuk98@framework:~/OpenVehicleDiag]$ cp common/Cargo.lock ~/nixos-config/packages/openvehiclediag/common.Cargo.lock
```

After that, it was a matter of updating `rev` and `hash`es:

```nix
{
  # parameters
}:
let
  # rootDir, version, and lockfile
in
assert builtins.elem program [
  "openvehiclediag"
  "cbf_parser"
];
rustPlatform.buildRustPackage (finalAttrs: {
  # package metadata

  src = fetchFromGitHub {
    owner = "rnd-ash";
    repo = "OpenVehicleDiag";
    rev = "0d9a6bdf88e219a6b9375b45a638cc5acc2fbcf3";
    hash = "sha256-SNX8CloGyk+UhZakhX5BuDvefzlvSrwzMrhuNtcqqmY=";
  };

  # patches, build inputs, and Cargo properties

  cargoDeps =
    if (program == "openvehiclediag") then
      (rustPlatform.fetchCargoVendor {
        src = finalAttrs.src;
        postPatch = finalAttrs.postPatch;
        cargoRoot = finalAttrs.cargoRoot;

        hash = "sha256-J7DonVSofVLixGkp7GaeJv/xZI+l+KenBWFXGjhNtYA=";
      })
    else
      null;
})
```

But the error was just about the same. With the failure, the J2534 protocol implementation was set to point to `3d14767`, [the second to the last commit](https://github.com/rnd-ash/J2534-Rust/commit/3d14767cd2f3a766cd2eb9accd041acbfde202b2 "Add CAN_MIXED_FORMAT IOCTL · rnd-ash/J2534-Rust@3d14767") at the time of writing. I was disappointed having to update the lockfile again.

```
[lyuk98@framework:~/OpenVehicleDiag]$ git restore .
[lyuk98@framework:~/OpenVehicleDiag]$ sed --in-place 's/branch="main"/rev="3d14767cd2f3a766cd2eb9accd041acbfde202b2"/g' app_rust/Cargo.toml
[lyuk98@framework:~/OpenVehicleDiag]$ cargo generate-lockfile --manifest-path app_rust/Cargo.toml
[lyuk98@framework:~/OpenVehicleDiag]$ cp app_rust/Cargo.lock ~/nixos-config/packages/openvehiclediag/openvehiclediag.Cargo.lock
```

The Cargo-related details were updated for the last time.

```nix
{
  # parameters
}:
let
  # rootDir, version, and lockfile
in
assert builtins.elem program [
  "openvehiclediag"
  "cbf_parser"
];
rustPlatform.buildRustPackage (finalAttrs: {
  # package metadata and source declaration

  postPatch = ''
    substituteInPlace ${rootDir.openvehiclediag}/Cargo.toml \
      --replace-fail 'branch="main"' 'rev="3d14767cd2f3a766cd2eb9accd041acbfde202b2"'

    ln --symbolic --verbose ${lockfile.openvehiclediag} ${rootDir.openvehiclediag}/Cargo.lock
    ln --symbolic --verbose ${lockfile.cbf_parser} ${rootDir.cbf_parser}/Cargo.lock
    ln --symbolic --verbose ${lockfile.common} ${rootDir.common}/Cargo.lock
  '';

  # build inputs and Cargo details

  cargoDeps =
    if (program == "openvehiclediag") then
      (rustPlatform.fetchCargoVendor {
        src = finalAttrs.src;
        postPatch = finalAttrs.postPatch;
        cargoRoot = finalAttrs.cargoRoot;

        hash = "sha256-SZXRA3N1WjnFH2IqRn+jl3NmjsDCWs/PZrRgh9/Du4Q=";
      })
    else
      null;
})
```

Attempting to build the software now resulted in errors and lots of warnings.

```
[lyuk98@framework:~/nixos-config]$ nix build .#openvehiclediag
error: Cannot build '/nix/store/47bknkzjjny01v8bi1mnyqbl5sprx9ck-openvehiclediag-1.0.5.drv'.
       Reason: builder failed with exit code 101.
       Output paths:
         /nix/store/6psh8bakv5zlx3h3c30kn3d4rkzgicfg-openvehiclediag-1.0.5
       Last 25 log lines:
       >     |
       > 447 |         ret_res(unsafe { (&self.set_prog_v_fn)(dev_id, pin, voltage) }, ())
       >     |                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
       >     |
       > note: delayed at compiler/rustc_hir_typeck/src/callee.rs:162:39 - disabled backtrace
       >    --> src/passthru.rs:447:26
       >     |
       > 447 |         ret_res(unsafe { (&self.set_prog_v_fn)(dev_id, pin, voltage) }, ())
       >     |                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
       >
       > note: we would appreciate a bug report: https://github.com/rust-lang/rust/issues/new?labels=C-bug%2C+I-ICE%2C+T-compiler&template=ice.md
       >
       > note: rustc 1.89.0 (29483883e 2025-08-04) (built from a source tarball) running on x86_64-unknown-linux-gnu
       >
       > note: compiler flags: --crate-type bin -C opt-level=3 -C embed-bitcode=no -C linker=/nix/store/x8mydcgbry214s802nzvy7fdljx404ym-gcc-wrapper-14.3.0/bin/cc -C force-frame-pointers=yes -C link-arg=/build/source/target/x86_64-unknown-linux-gnu/release/deps/openvehiclediag_audit_data.o -C link-arg=-Wl,--undefined=AUDITABLE_VERSION_INFO
       >
       > note: some of the compiler flags provided by cargo are hidden
       >
       > query stack during panic:
       > end of query stack
       > warning: `openvehiclediag` (bin "openvehiclediag") generated 133 warnings (run `cargo fix --bin "openvehiclediag"` to apply 14 suggestions)
       > error: could not compile `openvehiclediag` (bin "openvehiclediag"); 147 warnings emitted
       >
       > Caused by:
       >   process didn't exit successfully: `/nix/store/jbax1j60561siini5xyfs6j8sb32slvd-cargo-auditable-0.6.5/bin/cargo-auditable rustc --crate-name openvehiclediag --edition=2018 src/main.rs --error-format=json --json=diagnostic-rendered-ansi,artifacts,future-incompat --crate-type bin --emit=dep-info,link -C opt-level=3 -C embed-bitcode=no --check-cfg 'cfg(docsrs,test)' --check-cfg 'cfg(feature, values())' -C metadata=74848cf12507952e -C extra-filename=-04c7b5d57e294454 --out-dir /build/source/target/x86_64-unknown-linux-gnu/release/deps --target x86_64-unknown-linux-gnu -C linker=/nix/store/x8mydcgbry214s802nzvy7fdljx404ym-gcc-wrapper-14.3.0/bin/cc -L dependency=/build/source/target/x86_64-unknown-linux-gnu/release/deps -L dependency=/build/source/target/release/deps --extern backtrace=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libbacktrace-430034263cb4d78c.rlib --extern bitfield=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libbitfield-76beb140a4b39c7d.rlib --extern chrono=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libchrono-a5d34cac2bc1b03c.rlib --extern common=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libcommon-4820c60ed00e30b8.rlib --extern dialog=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libdialog-8648ad858310e979.rlib --extern hex=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libhex-8d8a08a1e0f80e93.rlib --extern hex_serde=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libhex_serde-359de23570d35ddc.rlib --extern iced=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libiced-b533341ccd242322.rlib --extern iced_graphics=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libiced_graphics-d1f27440203b546e.rlib --extern iced_native=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libiced_native-b6b17b68f0f1a5be.rlib --extern iced_wgpu=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libiced_wgpu-1c0d5f228c7afced.rlib --extern image=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libimage-5a177d76d484f02b.rlib --extern j2534_rust=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libj2534_rust-65579d95b0623cf1.rlib --extern lazy_static=/build/source/target/x86_64-unknown-linux-gnu/release/deps/liblazy_static-8b052b5a067f16a0.rlib --extern libc=/build/source/target/x86_64-unknown-linux-gnu/release/deps/liblibc-44f53aef7f436742.rlib --extern libloading=/build/source/target/x86_64-unknown-linux-gnu/release/deps/liblibloading-b821adddbb850f2d.rlib --extern nfd=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libnfd-c98c2f9caef77d8e.rlib --extern serde=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libserde-ad5f59aa4fd2034c.rlib --extern serde_derive=/build/source/target/release/deps/libserde_derive-e2d353062b6f65f1.so --extern serde_json=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libserde_json-dd932975777bbbdd.rlib --extern shellexpand=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libshellexpand-57f2ab015b5f11de.rlib --extern socketcan=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libsocketcan-a153cfa8ed1a7508.rlib --extern socketcan_isotp=/build/source/target/x86_64-unknown-linux-gnu/release/deps/libsocketcan_isotp-c53d501817fc1c84.rlib -Cforce-frame-pointers=yes -L native=/build/source/target/x86_64-unknown-linux-gnu/release/build/spirv_cross-82727506d2dcfb99/out -L native=/build/source/target/x86_64-unknown-linux-gnu/release/build/nfd-0865288f809fd3b2/out -L native=/nix/store/psvw7vs0h8kfggxjxlscsv8hl9189768-freetype-2.13.3/lib -L native=/nix/store/37pqcmv5hn278m1anryngfnia3862iqc-fontconfig-2.17.1-lib/lib -L native=/nix/store/psvw7vs0h8kfggxjxlscsv8hl9189768-freetype-2.13.3/lib -L native=/nix/store/86f8ip7shrdrrhwc12gqkspzn9vsd58m-expat-2.7.3/lib` (exit status: 101)
       For full logs, run:
         nix log /nix/store/47bknkzjjny01v8bi1mnyqbl5sprx9ck-openvehiclediag-1.0.5.drv
```

The problem, as it turned out, was that the code was apparently using "target dependent calling convention" on [targets](https://doc.rust-lang.org/stable/rustc/platform-support.html "Platform Support - The rustc book") (`x86_64-unknown-linux-gnu` for my device) that do not support the convention.

[One problematic source file](https://github.com/rnd-ash/OpenVehicleDiag/blob/8e8a9de7ab2d7874b3c630c8e029c4fd3f0932dc/app_rust/src/passthru.rs "OpenVehicleDiag/app_rust/src/passthru.rs at 8e8a9de7ab2d7874b3c630c8e029c4fd3f0932dc · rnd-ash/OpenVehicleDiag") contained a bunch of `extern "stdcall"`, which all led to warnings like the following:

```
warning: the calling convention "stdcall" is not supported on this target
  --> src/passthru.rs:23:5
   |
23 |     unsafe extern "stdcall" fn(name: *const libc::c_void, device_id: *mut u32) -> i32;
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = warning: this was previously accepted by the compiler but is being phased out; it will become a hard error in a future release!
   = note: for more information, see issue #130260 <https://github.com/rust-lang/rust/issues/130260>
   = note: `#[warn(unsupported_fn_ptr_calling_conventions)]` on by default
```

...and eventually errors:

```
note: no errors encountered even though delayed bugs were created

note: those delayed bugs will now be shown as internal compiler errors

error: internal compiler error: invalid abi for platform should have reported an error: "stdcall"
   --> src/passthru.rs:213:22
    |
213 |             unsafe { (&self.open_fn)(std::ptr::null() as *const libc::c_void, &mut id as *mut u32) };
    |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    |
note: delayed at compiler/rustc_hir_typeck/src/callee.rs:162:39 - disabled backtrace
   --> src/passthru.rs:213:22
    |
213 |             unsafe { (&self.open_fn)(std::ptr::null() as *const libc::c_void, &mut id as *mut u32) };
    |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

I admittedly do not have deep knowledge in unsafe Rust, but by reading [an issue that mentions this problem](https://github.com/rust-lang/rust/issues/130260 "Tracking issue for future-incompatibility lint `unsupported_fn_ptr_calling_conventions` · Issue #130260 · rust-lang/rust"), I had a feeling that those calling conventions can be removed. To test my theory, I added a patch script that removes all occurrences of `extern "stdcall"`.

```nix
{
  # parameters
}:
let
  # rootDir, version, and lockfile
in
assert builtins.elem program [
  "openvehiclediag"
  "cbf_parser"
];
rustPlatform.buildRustPackage (finalAttrs: {
  # package metadata and source declaration

  postPatch = ''
    substituteInPlace ${rootDir.openvehiclediag}/Cargo.toml \
      --replace-fail 'branch="main"' 'rev="3d14767cd2f3a766cd2eb9accd041acbfde202b2"'

    substituteInPlace ${rootDir.openvehiclediag}/src/passthru.rs \
      --replace-warn 'extern "stdcall" ' ""

    ln --symbolic --verbose ${lockfile.openvehiclediag} ${rootDir.openvehiclediag}/Cargo.lock
    ln --symbolic --verbose ${lockfile.cbf_parser} ${rootDir.cbf_parser}/Cargo.lock
    ln --symbolic --verbose ${lockfile.common} ${rootDir.common}/Cargo.lock
  '';

  # build inputs and Cargo details
})
```

I was glad that a lot of warnings and errors were gone, but I still faced an error.

```
[lyuk98@framework:~/nixos-config]$ nix build .#openvehiclediag
error: Cannot build '/nix/store/czpqwgwqp2qw3ncsgbsmgw0wx4kb0295-openvehiclediag-1.0.5.drv'.
       Reason: builder failed with exit code 101.
       Output paths:
         /nix/store/892b70w4mksyvilsdzwb4rc36qjn4y80-openvehiclediag-1.0.5
       Last 25 log lines:
       >    |
       > 28 |     pub fn new(comm_server: Box<dyn ComServer>, ecu: ISO15765Config) -> SessionResult<Self> {
       >    |                ^^^^^^^^^^^ help: if this is intentional, prefix it with an underscore: `_comm_server`
       >
       > warning: unused variable: `ecu`
       >   --> src/windows/diag_session/custom_session.rs:28:49
       >    |
       > 28 |     pub fn new(comm_server: Box<dyn ComServer>, ecu: ISO15765Config) -> SessionResult<Self> {
       >    |                                                 ^^^ help: if this is intentional, prefix it with an underscore: `_ecu`
       >
       > warning: unused variable: `msg`
       >   --> src/windows/diag_session/custom_session.rs:46:26
       >    |
       > 46 |     fn update(&mut self, msg: &Self::msg) -> Option<Self::msg> {
       >    |                          ^^^ help: if this is intentional, prefix it with an underscore: `_msg`
       >
       > warning: unused variable: `write_res`
       >   --> src/main.rs:48:13
       >    |
       > 48 |         let write_res = File::create(&path).unwrap().write_all(report.as_bytes());
       >    |             ^^^^^^^^^ help: if this is intentional, prefix it with an underscore: `_write_res`
       >
       > For more information about this error, try `rustc --explain E0061`.
       > warning: `openvehiclediag` (bin "openvehiclediag" test) generated 29 warnings
       > error: could not compile `openvehiclediag` (bin "openvehiclediag" test) due to 1 previous error; 29 warnings emitted
       For full logs, run:
         nix log /nix/store/czpqwgwqp2qw3ncsgbsmgw0wx4kb0295-openvehiclediag-1.0.5.drv
```

The problem originated from [what appeared to be a test](https://github.com/rnd-ash/OpenVehicleDiag/blob/main/app_rust/src/cli_tests/mod.rs "OpenVehicleDiag/app_rust/src/cli_tests/mod.rs at main · rnd-ash/OpenVehicleDiag").

```
error[E0061]: this function takes 5 arguments but 3 arguments were supplied
   --> src/cli_tests/mod.rs:38:22
    |
38  |            let server = KWP2000ECU::start_diag_session(
    |  _______________________^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^-
39  | |              api,
40  | |/             &ISO15765Config {
41  | ||                 baud: 500_000,
42  | ||                 send_id: 1460,
43  | ||                 recv_id: 1268,
...   ||
47  | ||                 use_ext_can: false,
48  | ||             },
    | ||_____________- expected `InterfaceType`, found `&ISO15765Config`
49  | |              None,
    | |              ---- argument #3 of type `InterfaceConfig` is missing
50  | |          )
    | |__________- argument #5 of type `DiagCfg` is missing
    |
note: expected `&Box<dyn ComServer>`, found `Box<dyn ComServer>`
   --> src/cli_tests/mod.rs:39:13
    |
39  |             api,
    |             ^^^
    = note: expected reference `&Box<(dyn ComServer + 'static)>`
                  found struct `Box<dyn ComServer>`
note: associated function defined here
   --> src/commapi/protocols/mod.rs:240:8
    |
240 |     fn start_diag_session(
    |        ^^^^^^^^^^^^^^^^^^
241 |         comm_server: &Box<dyn ComServer>,
    |         -----------
242 |         interface_type: InterfaceType,
    |         --------------
243 |         interface_cfg: InterfaceConfig,
    |         -------------
244 |         tx_flags: Option<Vec<PayloadFlag>>,
245 |         diag_cfg: DiagCfg,
    |         --------
help: consider borrowing here
    |
39  |             &api,
    |             +
help: provide the arguments
    |
38  -         let server = KWP2000ECU::start_diag_session(
39  -             api,
40  -             &ISO15765Config {
41  -                 baud: 500_000,
42  -                 send_id: 1460,
43  -                 recv_id: 1268,
44  -                 block_size: 8,
45  -                 sep_time: 20,
46  -                 use_ext_isotp: false,
47  -                 use_ext_can: false,
48  -             },
49  -             None,
50  -         )
38  +         let server = KWP2000ECU::start_diag_session(/* &Box<(dyn ComServer + 'static)> */, /* Interf>
    |
```

<s>Since I believe everything is still going to be fine,</s> I decided to remove it from the build process.

```nix
{
  # parameters
}:
let
  # rootDir, version, and lockfile
in
assert builtins.elem program [
  "openvehiclediag"
  "cbf_parser"
];
rustPlatform.buildRustPackage (finalAttrs: {
  # package metadata and source declaration

  postPatch = ''
    substituteInPlace ${rootDir.openvehiclediag}/Cargo.toml \
      --replace-fail 'branch="main"' 'rev="3d14767cd2f3a766cd2eb9accd041acbfde202b2"'

    substituteInPlace ${rootDir.openvehiclediag}/src/passthru.rs \
      --replace-warn 'extern "stdcall" ' ""
    substituteInPlace ${rootDir.openvehiclediag}/src/main.rs \
      --replace-warn 'mod cli_tests;' ""

    ln --symbolic --verbose ${lockfile.openvehiclediag} ${rootDir.openvehiclediag}/Cargo.lock
    ln --symbolic --verbose ${lockfile.cbf_parser} ${rootDir.cbf_parser}/Cargo.lock
    ln --symbolic --verbose ${lockfile.common} ${rootDir.common}/Cargo.lock
  '';

  # build inputs and Cargo details
})
```

## Fixing runtime errors

The build succeeded. I tried running the software right away, only to be disappointed again by a runtime error.

```
[lyuk98@framework:~/nixos-config]$ nix run .#openvehiclediag
Error: GraphicsAdapterNotFound
```

[iced](https://iced.rs/ "iced - A cross-platform GUI library for Rust"), the GUI library this program depends on, apparently needs [some tweaks](https://github.com/iced-rs/iced/issues/677#issuecomment-756287334 "Error: GraphicsAdapterNotFound · Issue #677 · iced-rs/iced") to get itself running on NixOS. I referred to Nixpkgs to find out how it is done [when creating a derivation](https://github.com/NixOS/nixpkgs/blob/938619092f449958852a01629069ade61c2555b6/pkgs/by-name/ha/halloy/package.nix "nixpkgs/pkgs/by-name/ha/halloy/package.nix at 938619092f449958852a01629069ade61c2555b6 · NixOS/nixpkgs").

After adding some more dependencies and changed some parts, this was my next attempt:

```nix
{
  fetchFromGitHub,
  lib,
  rustPlatform,
  stdenv,

  gtk3,
  libxkbcommon,
  pkg-config,
  vulkan-loader,
  wayland,
  xorg,

  program ? "openvehiclediag",
}:
let
  rootDir = {
    "openvehiclediag" = "app_rust";
    "cbf_parser" = "CBFParser";
    "common" = "common";
  };
  version = {
    "openvehiclediag" = "1.0.5";
    "cbf_parser" = "0.1.0";
    "common" = "0.1.0";
  };
  lockfile = {
    "openvehiclediag" = ./openvehiclediag.Cargo.lock;
    "cbf_parser" = ./cbf_parser.Cargo.lock;
    "common" = ./common.Cargo.lock;
  };
in
assert builtins.elem program [
  "openvehiclediag"
  "cbf_parser"
];
rustPlatform.buildRustPackage (
  finalAttrs:
  (
    {
      pname = program;
      version = version.${program};

      meta =
        let
          description = {
            openvehiclediag = "Cross-platform ECU diagnostics and car hacking application";
            cbf_parser = "Parses Mercedes CBF Files into OpenVehicleDiag's JSON";
          };
          longDescription = {
            openvehiclediag = ''
              Open Vehicle Diagnostics (OVD) is a Rust-based open source vehicle ECU diagnostic
              platform that makes use of the J2534-2 protocol, as well as SocketCAN on Linux!
            '';
            cbf_parser = ''
              This program converts Daimler CBF Files to the OVD JSON Schema. It can also be used
              to translate all the German strings in the CBF to a language of your choosing!
            '';
          };
        in
        {
          description = description.${program};
          longDescription = longDescription.${program};
          homepage = "https://github.com/rnd-ash/OpenVehicleDiag";
          changelog = "https://github.com/rnd-ash/OpenVehicleDiag/releases/tag/v${version.openvehiclediag}";
          license = lib.licenses.gpl3;
          mainProgram = program;
        };

      src = fetchFromGitHub {
        owner = "rnd-ash";
        repo = "OpenVehicleDiag";
        rev = "0d9a6bdf88e219a6b9375b45a638cc5acc2fbcf3";
        hash = "sha256-SNX8CloGyk+UhZakhX5BuDvefzlvSrwzMrhuNtcqqmY=";
      };

      postPatch = ''
        substituteInPlace ${rootDir.openvehiclediag}/Cargo.toml \
          --replace-fail 'branch="main"' 'rev="3d14767cd2f3a766cd2eb9accd041acbfde202b2"'

        substituteInPlace ${rootDir.openvehiclediag}/src/passthru.rs \
          --replace-warn 'extern "stdcall" ' ""
        substituteInPlace ${rootDir.openvehiclediag}/src/main.rs \
          --replace-warn 'mod cli_tests;' ""

        ln --symbolic --verbose ${lockfile.openvehiclediag} ${rootDir.openvehiclediag}/Cargo.lock
        ln --symbolic --verbose ${lockfile.cbf_parser} ${rootDir.cbf_parser}/Cargo.lock
        ln --symbolic --verbose ${lockfile.common} ${rootDir.common}/Cargo.lock
      '';

      cargoRoot = rootDir.${program};
      cargoLock.lockFile = lockfile.${program};
      buildAndTestSubdir = finalAttrs.cargoRoot;
    }
    // (lib.optionalAttrs (program == "openvehiclediag") {
      buildInputs = [
        gtk3
      ]
      ++ (lib.optionals stdenv.hostPlatform.isLinux [
        libxkbcommon
        vulkan-loader
        wayland
        xorg.libX11
        xorg.libxcb
        xorg.libXcursor
        xorg.libXi
        xorg.libXrandr
      ]);
      nativeBuildInputs = [
        pkg-config
      ];

      cargoDeps = rustPlatform.fetchCargoVendor {
        src = finalAttrs.src;
        postPatch = finalAttrs.postPatch;
        cargoRoot = finalAttrs.cargoRoot;

        hash = "sha256-SZXRA3N1WjnFH2IqRn+jl3NmjsDCWs/PZrRgh9/Du4Q=";
      };

      postFixup =
        let
          rpath = lib.makeLibraryPath [
            libxkbcommon
            vulkan-loader
            wayland
            xorg.libX11
          ];
        in
        lib.optionalString stdenv.hostPlatform.isLinux ''
          patchelf $out/bin/${program} \
            --add-rpath ${rpath}
        '';
    })
  )
)
```

The next build ran, but I could not see anything but a concerning message.

```
[lyuk98@framework:~/nixos-config]$ nix run .#openvehiclediag
interface 'wl_surface' has no event 2
```

I felt this application was not designed with Wayland in mind. To achieve a fallback to Xwayland, I referred to [ArchWiki](https://wiki.archlinux.org/title/Wayland#winit "Wayland - ArchWiki"), which described two methods of doing it:

> Winit is a window handling library in Rust. It will default to the Wayland backend, but it is possible to override it to Xwayland by modifying environment variables:
>
> - Prior to version 0.29.2, set `WINIT_UNIX_BACKEND=x11`
> - For version 0.29.2 and higher, unset `WAYLAND_DISPLAY`, which forces a fallback to X using the `DISPLAY` variable.

Either of them worked, so I added a wrapper doing both. The `postFixup` phase comes after `postInstall`, and `wrapProgram` replaces the binary with a shell script while moving the actual executable somewhere else; `patchelf` was therefore set to modify the now-hidden file.

```nix
{
  # parameters
}:
let
  rootDir, version, and lockfile
in
assert builtins.elem program [
  "openvehiclediag"
  "cbf_parser"
];
rustPlatform.buildRustPackage (
  finalAttrs:
  (
    {
      # package details
    }
    // (lib.optionalAttrs (program == "openvehiclediag") {
      # GUI app-specific details

      postInstall = lib.optionalString stdenv.hostPlatform.isLinux ''
        wrapProgram $out/bin/${program} \
          --unset WAYLAND_DISPLAY \
          --set WINIT_UNIX_BACKEND x11
      '';

      postFixup =
        let
          rpath = lib.makeLibraryPath [
            libxkbcommon
            vulkan-loader
            wayland
            xorg.libX11
          ];
        in
        lib.optionalString stdenv.hostPlatform.isLinux ''
          patchelf $out/bin/.${program}-wrapped \
            --add-rpath ${rpath}
        '';
    })
  )
)
```

## Adding some final details

Now that the application was running, one more thing I needed was a [desktop entry](https://specifications.freedesktop.org/desktop-entry-spec/latest/ "Desktop Entry Specification"). Together with an icon, the application was now ready to be launched by desktop managers.

```nix
{
  copyDesktopItems,
  fetchFromGitHub,
  lib,
  makeDesktopItem,
  makeWrapper,
  rustPlatform,
  stdenv,

  gtk3,
  libxkbcommon,
  pkg-config,
  vulkan-loader,
  wayland,
  xorg,

  program ? "openvehiclediag",
}:
let
  rootDir = {
    "openvehiclediag" = "app_rust";
    "cbf_parser" = "CBFParser";
    "common" = "common";
  };
  version = {
    "openvehiclediag" = "1.0.5";
    "cbf_parser" = "0.1.0";
    "common" = "0.1.0";
  };
  lockfile = {
    "openvehiclediag" = ./openvehiclediag.Cargo.lock;
    "cbf_parser" = ./cbf_parser.Cargo.lock;
    "common" = ./common.Cargo.lock;
  };

  description = {
    openvehiclediag = "Cross-platform ECU diagnostics and car hacking application";
    cbf_parser = "Parses Mercedes CBF Files into OpenVehicleDiag's JSON";
  };
  longDescription = {
    openvehiclediag = ''
      Open Vehicle Diagnostics (OVD) is a Rust-based open source vehicle ECU diagnostic
      platform that makes use of the J2534-2 protocol, as well as SocketCAN on Linux!
    '';
    cbf_parser = ''
      This program converts Daimler CBF Files to the OVD JSON Schema. It can also be used
      to translate all the German strings in the CBF to a language of your choosing!
    '';
  };
in
assert builtins.elem program [
  "openvehiclediag"
  "cbf_parser"
];
rustPlatform.buildRustPackage (
  finalAttrs:
  (
    {
      pname = program;
      version = version.${program};

      meta = {
        description = description.${program};
        longDescription = longDescription.${program};
        homepage = "https://github.com/rnd-ash/OpenVehicleDiag";
        changelog = "https://github.com/rnd-ash/OpenVehicleDiag/releases/tag/v${version.openvehiclediag}";
        license = lib.licenses.gpl3;
        mainProgram = program;
      };

      src = fetchFromGitHub {
        owner = "rnd-ash";
        repo = "OpenVehicleDiag";
        rev = "0d9a6bdf88e219a6b9375b45a638cc5acc2fbcf3";
        hash = "sha256-SNX8CloGyk+UhZakhX5BuDvefzlvSrwzMrhuNtcqqmY=";
      };

      postPatch = ''
        substituteInPlace ${rootDir.openvehiclediag}/Cargo.toml \
          --replace-fail 'branch="main"' 'rev="3d14767cd2f3a766cd2eb9accd041acbfde202b2"'

        substituteInPlace ${rootDir.openvehiclediag}/src/passthru.rs \
          --replace-warn 'extern "stdcall" ' ""
        substituteInPlace ${rootDir.openvehiclediag}/src/main.rs \
          --replace-warn 'mod cli_tests;' ""

        ln --symbolic --verbose ${lockfile.openvehiclediag} ${rootDir.openvehiclediag}/Cargo.lock
        ln --symbolic --verbose ${lockfile.cbf_parser} ${rootDir.cbf_parser}/Cargo.lock
        ln --symbolic --verbose ${lockfile.common} ${rootDir.common}/Cargo.lock
      '';

      cargoRoot = rootDir.${program};
      cargoLock.lockFile = lockfile.${program};
      buildAndTestSubdir = finalAttrs.cargoRoot;
    }
    // (lib.optionalAttrs (program == "openvehiclediag") {
      buildInputs = [
        gtk3
      ]
      ++ (lib.optionals stdenv.hostPlatform.isLinux [
        libxkbcommon
        vulkan-loader
        wayland
        xorg.libX11
        xorg.libxcb
        xorg.libXcursor
        xorg.libXi
        xorg.libXrandr
      ]);
      nativeBuildInputs = [
        copyDesktopItems
        makeWrapper
        pkg-config
      ];

      cargoDeps = rustPlatform.fetchCargoVendor {
        src = finalAttrs.src;
        postPatch = finalAttrs.postPatch;
        cargoRoot = finalAttrs.cargoRoot;

        hash = "sha256-SZXRA3N1WjnFH2IqRn+jl3NmjsDCWs/PZrRgh9/Du4Q=";
      };

      desktopItems = builtins.map makeDesktopItem [
        {
          name = program;
          desktopName = "Open Vehicle Diagnostics";
          comment = description.${program};
          icon = "rand_ash-${program}";
          exec = finalAttrs.pname;
          terminal = false;
          categories = [
            "Utility"
          ];
        }
      ];

      postInstall = lib.optionalString stdenv.hostPlatform.isLinux ''
        install -D --mode 0644 \
          app_rust/img/launcher.png \
          $out/share/icons/hicolor/256x256/apps/rand_ash-${program}.png

        wrapProgram $out/bin/${program} \
          --unset WAYLAND_DISPLAY \
          --set WINIT_UNIX_BACKEND x11
      '';

      postFixup =
        let
          rpath = lib.makeLibraryPath [
            libxkbcommon
            vulkan-loader
            wayland
            xorg.libX11
          ];
        in
        lib.optionalString stdenv.hostPlatform.isLinux ''
          patchelf $out/bin/.${program}-wrapped \
            --add-rpath ${rpath}
        '';
    })
  )
)
```

## The result

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/a39bea90-56e0-4349-b2dd-89753bc85d39.avif">
  <img src="https://images.lyuk98.com/b38ad111-48f0-466d-8d93-b5bfee77cb3a.avif" alt="The main window of Open Vehicle Diagnostics" title="The result">
</picture>

The application was running, but I was not able to test anything further since I do not currently own [a compatible adaptor](https://github.com/rnd-ash/OpenVehicleDiag/tree/0d9a6bdf88e219a6b9375b45a638cc5acc2fbcf3/app_rust#platform-support "OpenVehicleDiag/app_rust at 0d9a6bdf88e219a6b9375b45a638cc5acc2fbcf3 · rnd-ash/OpenVehicleDiag") yet. Working on it, though, alongside with fixing Friction, would be fun.

# Finishing up

It was my first time to properly package something. I liked that Nix derivations aim to create deterministic builds by default; I also did not have to worry about having to downgrade existing packages in my system to match the dependencies, like what I might have done with FFmpeg.

However, it was not without downsides. I have referred to documentations to reach the solution I ended up with, but they were sort of scattered everywhere and were sometimes difficult to locate. Here is a non-exhaustive list of references I have come across during the process:

- [Nixpkgs Reference Manual](https://nixos.org/manual/nixpkgs/stable/ "Nixpkgs Reference Manual")
- [Noogle](https://noogle.dev/ "Noogle - Simply find Nix API reference documentation.")
- [NixOS Wiki](https://wiki.nixos.org/wiki/NixOS_Wiki "NixOS Wiki - NixOS Wiki")

Let us go back to when I wanted to set and unset runtime environment variables for Open Vehicle Diagnostics. I cannot simply set them within the derivation's attribute set, because they are not passed to the executable.

```nix
{ rustPlatform }: rustPlatform.buildRustPackage {
  # These are only recognised during build time, not runtime
  WAYLAND_DISPLAY = null; # how to unset this?
  WINIT_UNIX_BACKEND = "x11";
}
```

`runtimeEnv` does exist, but only for creating a shell application using `writeShellApplication`:

```nix
{ rustPlatform }: rustPlatform.buildRustPackage {
  # This will complain that an attribute set cannot be coerced into a string
  runtimeEnv = {
    WAYLAND_DISPLAY = null; # how to unset this?
    WINIT_UNIX_BACKEND = "x11";
  };
}
```

The solution was [to wrap the program](https://nixos.org/manual/nixpkgs/stable/#fun-makeWrapper "Nixpkgs Reference Manual") with `makeWrapper`. By looking at an example that has been presented to me, it seemed like I can set environment variables with it.

> ```sh
> # adds `FOOBAR=baz` to `$out/bin/foo`’s environment
> makeWrapper $out/bin/foo $wrapperfile --set FOOBAR baz
> ```

But what about the part where I want to *unset* one? I guess `--unset` makes sense, but where is any documentation to confirm this?

It took a while to find what I have been looking for, which was hidden [in the code](https://github.com/NixOS/nixpkgs/blob/7b9c6346ffc9c16910c25c56534733e70935f306/pkgs/build-support/setup-hooks/make-wrapper.sh#L13-L39 "nixpkgs/pkgs/build-support/setup-hooks/make-wrapper.sh at 7b9c6346ffc9c16910c25c56534733e70935f306 · NixOS/nixpkgs"):

> ```sh
> # ARGS:
> # --argv0        NAME    : set the name of the executed process to NAME
> #                          (if unset or empty, defaults to EXECUTABLE)
> # --inherit-argv0        : the executable inherits argv0 from the wrapper.
> #                          (use instead of --argv0 '$0')
> # --resolve-argv0        : if argv0 doesn't include a / character, resolve it against PATH
> # --set          VAR VAL : add VAR with value VAL to the executable's environment
> # --set-default  VAR VAL : like --set, but only adds VAR if not already set in
> #                          the environment
> # --unset        VAR     : remove VAR from the environment
> # --chdir        DIR     : change working directory (use instead of --run "cd DIR")
> # --run          COMMAND : run command before the executable
> # --add-flag     ARG     : prepend the single argument ARG to the invocation of the executable
> #                          (that is, *before* any arguments passed on the command line)
> # --append-flag  ARG     : append the single argument ARG to the invocation of the executable
> #                          (that is, *after* any arguments passed on the command line)
> # --add-flags    ARGS    : prepend ARGS verbatim to the Bash-interpreted invocation of the executable
> # --append-flags ARGS    : append ARGS verbatim to the Bash-interpreted invocation of the executable
> 
> # --prefix          ENV SEP VAL   : suffix/prefix ENV with VAL, separated by SEP
> # --suffix
> # --prefix-each     ENV SEP VALS  : like --prefix, but VALS is a list
> # --suffix-each     ENV SEP VALS  : like --suffix, but VALS is a list
> # --prefix-contents ENV SEP FILES : like --suffix-each, but contents of FILES
> #                                   are read first and used as VALS
> # --suffix-contents
> makeWrapper() { makeShellWrapper "$@"; }
> ```

Even before properly knowing about Nix and NixOS, I have seen people saying that they have a high learning curve. I am in the process of making them my second nature, and I think I am doing pretty good about it so far; however, I still felt a problem like this could deter potential contributors into the Nix ecosystem.

Despite the shortcomings, however, I do not think I can go back to other Linux distributions (not for now, at least). Declarative management of resources for a whole operating system is something I did not know I needed, and the deterministic nature of derivations makes me a little more confident that changes to the system will not affect the application.
