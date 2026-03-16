---
date: '2025-12-25'
title: Using Zrythm for audio production
description: I gave Zrythm a try for recording music before the end of this year.
---

Sometimes, I like to record music and videos that will be left as pieces of memory to reminisce about. I am neither a professional music producer nor a content creator, but I have managed to learn just enough to do what I wanted.

As I worked on another one before finishing this year, I wanted to try a new tool. As one could guess from the title, I gave [Zrythm](https://www.zrythm.org/ "Zrythm - Digital Audio Workstation") a shot.

---

# Switching to Zrythm

![A screenshot of the Zrythm digital audio workstation](https://images.lyuk98.com/791bcbe6-dede-4630-990f-a8f8bf3a2947.avif)

> Zrythm is a digital audio workstation tailored for both professionals and beginners, offering an intuitive interface and robust functionality.

I doubt many people use Linux for audio production, but I happened to be one of them that do. [Ardour](https://ardour.org/ "Ardour, free and open-source digital audio workstation | Ardour DAW") has been my choice of tool for a while, but one thing always bothered me: its use of GTK 2, which [reached end of its life](https://blog.gtk.org/2020/12/16/gtk-4-0/#:~:text=GTK%202%20has%20reached%20the%20end%20of%20its%20life "GTK 4.0 – GTK Development Blog") several years ago. Granted, it uses (and now [mandates the use of](https://github.com/Ardour/ardour/commit/1e28ee9cc980f9f93734a2022f04cf2298b2091d "YTK is no longer optional · Ardour/ardour@1e28ee9")) its own localised version of the long-obsolete library called YTK, but it just felt like delaying the eventual need to migrate to a newer technology. It does work fine right now, but I wanted to learn other ways to achieve my goal.

That is where [Zrythm](https://www.zrythm.org/ "Zrythm - Digital Audio Workstation") came in. It is currently undergoing [a major rewrite](https://forum.zrythm.org/t/some-updates-on-v1-release-and-v2-development "Some updates on v1 release (and v2 development) - News - Zrythm Forum") from C to C++ and from GTK 4 to Qt 6, but I still decided to give its GTK-based stable version ([v1.0.0](https://forum.zrythm.org/t/release-announcement-zrythm-v1-0-0/201 "Release Announcement: Zrythm v1.0.0 - News - Zrythm Forum") at the time of writing) a go.

# Creating a project and configuring the system

A simple window showed up as I started the application. I went ahead and created a new project.

![A dialog for choosing and opening a project in Zrythm. The list of recent projects is empty, and two buttons with contents "Create New Project..." and "Open From Path..." are present.](https://images.lyuk98.com/4478f1e8-31a7-4679-b78f-426e58dbe2d9.avif)

After entering some details, a new project was created.

![A dialog for creating a new project in Zrythm. "Project Name" is set to "Untitled Project", "Parent Directory" as "~/.local/share/zrythm/projects", and "Template" as "Blank project".](https://images.lyuk98.com/06a4eaa2-5b04-4bfc-a1f8-445f0732279b.avif)

Two errors greeted me as I was about to get started, however.

![An error message that says: "Your user does not have enough privileges to allow Zrythm to set the scheduling priority of threads. Please refer to the 'Getting Started' section in the user manual for details."](https://images.lyuk98.com/75a5918b-384d-4c9b-9b02-8de2c1a358ec.avif)![An error message that says: "Your user does not have enough privileges to allow Zrythm to lock unlimited memory. This may cause audio dropouts. Please refer to the 'Getting Started' section in the user manual for details."](https://images.lyuk98.com/000f8967-91ba-4dbf-9db9-645037af5bed.avif)

My system uses [PipeWire](https://pipewire.org/ "PipeWire") as the sound server and, fortunately, all I had to do to enjoy [the increased limits](https://github.com/NixOS/nixpkgs/blob/674c2b09c59a220204350ced584cadaacee30038/nixos/modules/services/desktops/pipewire/pipewire.nix#L421-L441 "nixpkgs/nixos/modules/services/desktops/pipewire/pipewire.nix at 674c2b09c59a220204350ced584cadaacee30038 · NixOS/nixpkgs") was to add my user to the group `pipewire`.

```nix
{ config, ... }:
{
  users.users.lyuk98.extraGroups =
    let
      # Retain only valid group names
      ifTheyExist = groups: builtins.filter (group: builtins.hasAttr group config.users.groups) groups;
    in
    ifTheyExist [
      "adbusers"
      "libvirtd"
      "networkmanager"
      "pipewire"
      "tss"
      "wheel"
    ];
}
```

One caveat, though, was that the limit on how much memory can be locked [was set to about 4 gigabytes](https://github.com/NixOS/nixpkgs/blob/674c2b09c59a220204350ced584cadaacee30038/nixos/modules/services/desktops/pipewire/pipewire.nix#L435-L440 "nixpkgs/nixos/modules/services/desktops/pipewire/pipewire.nix at 674c2b09c59a220204350ced584cadaacee30038 · NixOS/nixpkgs") instead of `unlimited`; I decided not to do anything for now, however, unless I come across any problem about it.

Lastly, I set the audio and Midi backend to Jack, which PipeWire will take care of.

![Preferences menu for Zrythm. "Audio backend" is set to "JACK", and "MIDI backend" is set to "JACK MIDI".](https://images.lyuk98.com/2ee12a38-67c9-4c24-9863-bee8efe555c1.avif)

# Adding audio plugins

There are three problems I have previously encountered during my recording sessions:

1. I do not (properly, at least) play drums, which I still wish to include in my recording.
2. I connect the instrument directly to my audio interface; it is much quieter than recording with a microphone in front of an amplifier, but it sounds terrible without some sort of amplifier emulation.
3. My guitar, having single-coil pickups, is susceptible to a great amount of hum; it especially stands out when some sort of distortion (such as fuzz) is involved in the signal chain.

My solution was to employ audio plugins: [DrumGizmo](https://drumgizmo.org/ "start – DrumGizmo Wiki"), [Guitarix](https://guitarix.org/ "Guitarix - GNU/Linux Virtual Amplifier"), and [Noise Repellent](https://github.com/lucianodato/noise-repellent "lucianodato/noise-repellent: Lv2 suite of plugins for broadband noise reduction"), which are [LV2](https://lv2plug.in/ "LV2") plugins. Because Zrythm could not find them under non-FHS-compliant NixOS, I copied and modified [a solution on NixOS Wiki](https://wiki.nixos.org/wiki/Audio_production#Plugins_not_found "Audio production - Official NixOS Wiki") to make them discoverable.

```nix
{
  pkgs,
  lib,
  config,
  ...
}:
{
  # Add environment variables for audio plugins
  home.sessionVariables =
    let
      makePluginPath =
        format:
        (lib.makeSearchPath format [
          "${config.home.profileDirectory}/lib"
          "/run/current-system/sw/lib"
        ])
        + ":${config.home.homeDirectory}/.${format}";
    in
    {
      DSSI_PATH = makePluginPath "dssi";
      LADSPA_PATH = makePluginPath "ladspa";
      LV2_PATH = makePluginPath "lv2";
      LXVST_PATH = makePluginPath "lxvst";
      VST_PATH = makePluginPath "vst";
      VST3_PATH = makePluginPath "vst3";
    };

  # Add audio plugins
  home.packages = with pkgs; [
    drumgizmo # DrumGizmo LV2 plugin
    guitarix # Guitarix
    gxplugins-lv2 # Extra LV2 plugins to complement Guitarix
    noise-repellent # LV2 plugins for noise reduction
  ];
}
```

I could then let the application find them by providing the paths as environment variables.

![Preferences menu for Zrythm. The "Paths" section is shown, which description says: "Multiple search paths must be separated by the : character. Environment variables are supported with the following syntax: ${ENV_VAR_NAME}. Leave blank to use default paths." Paths for "LV2 plugins" is set to "${LV2_PATH}", "VST2 plugins" as "${VST_PATH}", "DSSI plugins" as "${DSSI_PATH}", "LADSPA plugins" as "${LADSPA_PATH}", and "VST3 plugins" as "${VST3_PATH}". Paths for "SFZ instruments", "JSFX plugins", "CLAP plugins", and "SF2 instruments" are empty.](https://images.lyuk98.com/4c747a2e-2edb-4636-bbd5-3ee73a710b5e.avif)

# Recording tracks

It was time to record some tracks, while overcoming whatever challenges Zrythm may throw at me.

## Creating tracks for the drums

While DrumGizmo provides a way to play drums, it does not have drums themselves. A [compatible drumkit](https://drumgizmo.org/wiki/doku.php?id=kits "kits – DrumGizmo Wiki") is required, where I chose [DRSKit](https://drumgizmo.org/wiki/doku.php?id=kits:drskit "kits:drskit – DrumGizmo Wiki").

![A drumkit](https://images.lyuk98.com/073de58f-b3cc-4a2a-8056-ea638c28732d.avif)

To use the drumkit, I downloaded and extracted an archive.

```
[lyuk98@framework:~]$ curl --remote-name https://drumgizmo.org/kits/DRSKit/DRSKit2_1.zip
[lyuk98@framework:~]$ unzip DRSKit2_1.zip
```

The drumkit was ready, and I double-clicked the DrumGizmo plugin at the bottom right to create an instrument track.

![A list of available plugins in Zrythm. "DrumGizmo" is selected.](https://images.lyuk98.com/1e611f89-38b5-4eab-9d68-38232953b2a9.avif)

I was then asked to let Zrythm do some routing for me. I clicked "No" as I decided to do it myself.

![A dialog that says: "This plugin contains multiple audio outputs. Would you like to auto-route each output to a separate FX track?" Options "Yes", "No", and "Cancel" are available.](https://images.lyuk98.com/1512082d-993d-4004-a726-70f49264bd4a.avif)

The DrumGizmo's instrument UI showed up afterwards.

![The interface for the DrumGizmo instrument plugin](https://images.lyuk98.com/b013a4ff-f9f7-4a63-871b-bae728a91c30.avif)

Since I have previously used this plugin with Ardour, I was pretty familiar with its interface. I set drumkit and midimap files, and also tweaked a few settings to my preference.

![The interface for the DrumGizmo instrument plugin. Compared to the initial state, drumkit file was changed to "/home/lyuk98/DRSKit/DRSKit_full.xml", midimap file to "/home/lyuk98/DRSKit/Midimap_full.xml". Quality is increased to the maximum and cache limit is increased to unlimited.](https://images.lyuk98.com/280b64b3-8be7-46de-9a5f-25062c132d6b.avif)

"Timing Humanizer" remained off for now, since it introduces a noticeable delay during the recording session. I decided to give it a go before exporting the final audio.

### Routing audio

DrumGizmo provides 16 output channels; what they will be used for is up to each drumkit. For DRSKit, 13 of them are actually used, and what each channel is for [is documented on its web page](https://drumgizmo.org/wiki/doku.php?id=kits:drskit#channel_setup "kits:drskit – DrumGizmo Wiki").

> All microphones are connected to its own channel when loading the kit in DrumGizmo. 13 channels total. Remember to pan the relevant channels to give you a better stereo effect.
> 
> - Ch 1: Ambience left
> - Ch 2: Ambience right
> - Ch 3: Kickdrum back
> - Ch 4: Kickdrum front
> - Ch 5: Hihat
> - Ch 6: Overhead left
> - Ch 7: Overhead right
> - Ch 8: Ride cymbal
> - Ch 9: Snaredrum bottom
> - Ch 10: Snaredrum top
> - Ch 11: Tom1
> - Ch 12: Tom2 (Floor tom)
> - Ch 13: Tom3 (Floor tom)

With Ardour, I created separate tracks for each channel, connecting them within the application's interface.

| Tracks | Audio connections |
| --- | --- |
| ![Ardour's user interface. Under the "Drums" group, tracks "Ambience", "Kickdrum back", "Kickdrum front", "Hihat", "Overhead", "Ride cymbal", "Snaredrum bottom", "Snaredrum top", "Tom1", "Tom2 (Floor tom)", and "Tom3 (Floor tom)" are present.](https://images.lyuk98.com/265cfb36-df6e-4b3e-9f4c-034af5aa23ec.avif) | ![Ardour's audio connection manager. Channels 1 to 13 in "Drums out" are each connected to left and right tracks of "Ambience", "Kickdrum back", "Kickdrum front", "Hihat", left and right tracks of "Overhead", "Ride cymbal", "Snaredrum bottom", "Snaredrum top", "Tom1", "Tom2 (Floor tom)", and "Tom3 (Floor tom)", respectively.](https://images.lyuk98.com/1aba26cc-7e99-4463-bc6c-84de65a1bde4.avif) |

However, how it was done with Zrythm was quite different. I first added a [folder track](https://manual.zrythm.org/en/tracks/track-types.html#folder-track "Track Types - Zrythm v2.0.0-DEV documentation"), put the [instrument track](https://manual.zrythm.org/en/tracks/track-types.html#instrument-track "Track Types - Zrythm v2.0.0-DEV documentation") (from DrumGizmo) under it, and added an [audio group track](https://manual.zrythm.org/en/tracks/track-types.html#audio-group-track "Track Types - Zrythm v2.0.0-DEV documentation") with [audio FX tracks](https://manual.zrythm.org/en/tracks/track-types.html#audio-fx-track "Track Types - Zrythm v2.0.0-DEV documentation") representing each channel.

![Zrythm's user interface. Under the "Drums" folder track, an instrument track "Drums (Midi)" and an audio group track "Drums (Audio)" are present. Under the group track, audio FX tracks named "Ambience", "Kickdrum back", "Kickdrum front", "Hihat", left and right tracks of "Overhead", "Ride cymbal", "Snaredrum bottom", "Snaredrum top", "Tom1", "Tom2 (Floor tom)", and "Tom3 (Floor tom)" are present.](https://images.lyuk98.com/9ddb4d1c-0109-4c5f-9918-24299c6d9d53.avif)

After double-clicking the instrument track to see its properties, I went to inspect the plugin.

![Track properties of the track "Drums (Midi)". A context menu for the track's intrument plugin, DrumGizmo, is shown, where there are three options "Bypass", "Inspect", and "Change Load Behavior".](https://images.lyuk98.com/63daef22-ba96-4a42-97e5-b761b6d94f7a.avif)

From there, I could route each output to corresponding channels.

| Result | Port selection |
| --- | --- |
| ![The "Audio Outs" section of the track "Drums (Midi)". An output is selected for the entry "Audio out 0", which is set to "Ambience/TP Stereo in L"](https://images.lyuk98.com/b02e1072-f219-43cc-894b-b87cb49ba71a.avif) | ![The port selection interface in Zrythm. For "Track", "Plugin", and "Port", "Ambience", "Track Ports", and "TP Stereo in L" are selected, respectively.](https://images.lyuk98.com/bf9236af-4b38-462d-b86f-cf0c621e50dc.avif) |

I could not find out how to make tracks mono, so I simply added both left and right channels when needed.

![The "Audio Outs" section of the track "Drums (Midi)". Two outputs are selected for the entry "Audio out 2", which is set to "Kickdrum back/TP Stereo in L" and "Kickdrum back/TP Stereo in R"](https://images.lyuk98.com/dbc89076-c6a8-4497-b3fc-971b098edda7.avif)

### Entering notes and playing them back

Some notes were now in place, but I could not hear anything upon playback.

![The user interface for Zrythm, where some Midi notes for the track "Drums (Midi)" present](https://images.lyuk98.com/d7162c7c-0b9b-43e4-9dae-302aa0abe689.avif)

I have previously seen an error regarding Jack, and I thought something went wrong during audio routing, resulting in silence.

![An error message that says the following: "Failed to connect to left monitor output port - Details: - JACK: Failed to connect monitor output \[left\]: Overall operation failed"](https://images.lyuk98.com/82306a98-6c08-4dfa-b332-1d51eab5dd8d.avif)

My suspicion was that, because my system uses a custom PipeWire sink as part of [audio enhancement](https://github.com/NixOS/nixos-hardware/blob/9154f4569b6cdfd3c595851a6ba51bfaa472d9f3/framework/13-inch/common/audio.nix#L13-L37 "nixos-hardware/framework/13-inch/common/audio.nix at 9154f4569b6cdfd3c595851a6ba51bfaa472d9f3 · NixOS/nixos-hardware"), Zrythm could not route to the raw device. It could be solved by manually connecting them using [qpwgraph](https://gitlab.freedesktop.org/rncbc/qpwgraph "Rui Nuno Capela / qpwgraph · GitLab").

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/89a45247-d70b-44eb-8773-18559da18283.avif">
  <img src="https://images.lyuk98.com/cec44c4e-5c64-4ef8-8724-c4938419ce57.avif" alt="The interface for qpwgraph. Channels &quot;Master/Stereo out L&quot; and &quot;Monitor Out L&quot; from &quot;Zrythm&quot; is connected to the channel &quot;playback_FL&quot; from &quot;Framework Speakers&quot;, and channels &quot;Master/Stereo out R&quot; and &quot;Monitor Out R&quot; to the channel &quot;playback_FR&quot;.">
</picture>

The drums were now audible, but I noticed that the application labelled some notes incorrectly. I have previously used [Hydrogen](http://hydrogen-music.org/ "Hydrogen") with Ardour, but I decided to stick with Zrythm this time. As a consequence, I had to keep the midimap file around and continuously find numbers of the notes I wished to use.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://images.lyuk98.com/0809f7d1-c5ab-4c10-a4e1-48eb57122983.avif">
  <img src="https://images.lyuk98.com/494003f3-49b0-43f0-a951-a03e10f5b15c.avif" alt="The note editor for Zrythm, as well as the content of the midimap file">
</picture>

## Recording instruments

The Midi track was complete, and the next thing to do was to record a few audio tracks. Creating one was as easy as clicking `+` and selecting `Add Audio Track`. Adding a plugin could be done by dragging one to the `Inserts` section.

![The user interface of Zrythm. Notably, a new audio track named "Bass" is present and a plugin named "GxAmplifier-X" is present in the "Inserts" list of the track.](https://images.lyuk98.com/03cb9724-b02a-4ee7-88fa-0a874e7671b9.avif)

A problem, however, was that I could not figure out how to connect the instrument to the audio track, as fiddling with the `Inputs` section seemingly did not do anything. I later ended up once again relying on qpwgraph for routing.

![The interface for qpwgraph, where the channel "capture_FL" from "Studio 24c Digital Stereo" is connected to channels "Bass/TP Stereo in L" and "Bass/TP Stereo in R" from "Zrythm"](https://images.lyuk98.com/b0fd070d-af34-4310-8643-3c751f01fc58.avif)

The rest was similar to how it worked with Ardour.

| Arm for recording | Record |
| --- | --- |
| ![An audio track in Zrythm, with a tooltip "Arm for recording" shown](https://images.lyuk98.com/9a365b4a-4392-441e-a396-592f3353ecc5.avif) | ![The playback interface in Zrythm, with a tooltip "Record" shown](https://images.lyuk98.com/9aee3dfe-d3f3-4552-a060-98eb01dc1228.avif) |

## Exporting audio

After all the recording and editing, the project was ready to be exported. The process could be started from the top-right menu.

![Top-right hamburger menu in Zrythm. It is opened and highlights the option "Export As..."](https://images.lyuk98.com/1b47d62a-1f5a-4023-9245-aa0d20e95b78.avif)

The process began after modifying some options, and the audio was ready in a few minutes.

![A dialog for setting export options. There are four sections: "Metadata", "Options", "Selection", and "Output", where the first three have user-modifiable settings. The "Metadata" section has options "Title", "Artist", and "Genre", which are "My Project", "Zrythm", and "Electronic", respectively. For the "Options" section, "Format" is set to "FLAC", "Bit Depth" as "24 bit", "Dither" as disabled, "Filename Pattern" as "\<name\>.\<format\>", and "Mixdown or Stems" as "Mixdown". For the "Selection" section, "Time Range is set to "Loop" and the menu "Track Selection" is collapsed. For the "Output" section, it indicates that: "The following file will be created: mixdown.flac" "in the directory: /home/lyuk98/.local/share/zrythm/projects/Untitled Project/exports".](https://images.lyuk98.com/2e113eb8-a08f-4b66-abcd-c90fa282f17c.avif)![A dialog with the progress of the export process](https://images.lyuk98.com/0e1dd2a8-b679-447d-82cc-2063d66ebd71.avif)

However, as I mixed the audio somewhat quietly, it was not loud enough to listen to. Ardour provided a way to normalise audio during the export process, but Zrythm lacked such a functionality. To achieve the desired loudness, I ran [Audacity](https://www.audacityteam.org/ "Audacity ® | Free Audio editor, recorder, music making and more!") and performed "loudness normalization"

![Audacity's interface with an imported audio file shown](https://images.lyuk98.com/11938f5b-dd16-441b-b18c-48029affc86f.avif)![A dialog for adjusting the loudness normalisation process in Audacity. It is set to "normalize perceived loudness to -14.0 LUFS", the option "Normalize stereo channels independently" is unchecked, and the option "Treat mono as dual-mono (recommended)" is checked.](https://images.lyuk98.com/ca432367-6dd7-430b-954a-ec7bef8d39da.avif)

I also added some metadata to the audio later on, which was done with [Kid3](https://kid3.kde.org/ "Kid3 - Audio Tagger").

---

# Editing videos

While recording the project, I also recorded some videos that were to be paired with the audio. To do the video editing, I chose [DaVinci Resolve](https://www.blackmagicdesign.com/products/davinciresolve "DaVinci Resolve | Blackmagic Design").

![A splash screen of DaVinci Resolve](https://images.lyuk98.com/8a804cb2-8296-4e31-bfb5-f4f3ae41db93.avif)

I have long insisted on using open software, even if it led to difficulties and lack of support. However, in this case, I was not able to find free software that can properly deal with footages that use something other than the [Rec 709](https://www.itu.int/rec/R-REC-BT.709 "BT.709 : Parameter values for the HDTV standards for production and international programme exchange") standard, and given that the support for high-dynamic range itself in Wayland [has been merged](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/merge_requests/14 "staging: add color management protocol (!14) · Merge requests · wayland / wayland-protocols · GitLab") rather recently, I felt I do not have many choices to choose from.

---

# Finishing up

I am all-in for seeing free alternatives to popular proprietary software become actually viable and, eventually, widely adopted (and not just be "alternatives"). It would be great to see Zrythm go such a path, but in its current stable version, it is seemingly not there just yet.

One feature I wished to see was a proper tempo and time signature automation. The application technically supports it in the current version, but upon activation, I was warned of its experimental state.

![A message titled "Experimental Feature" that says: "BPM and time signature automation is an experimental feature. Using it may corrupt your project."](https://images.lyuk98.com/f1dcab98-18b4-42a7-b763-7e63b2f8ae05.avif)

The project I worked on had changes to time signature on certain parts, and unlike my expectation, Zrythm temporarily changed the time signature of the whole song (and not just the affected parts) when the playhead went over the automated area.

| <span class="music-text" aria-description="4/4 time signature"></span> | <span class="music-text" aria-description="3/4 time signature"></span> |
| --- | --- |
| ![The timeline view of Zrythm, where the time signature automation is set to gradually change beats per bar from 4 to 3, and 3 to 4 at bars 3 and 4. The playhead is at the start of bar 1 and the entire timeline grid shows four beats for each bar.](https://images.lyuk98.com/e58cb998-56f5-4f7e-84ae-cab74b02e18d.avif) | ![The timeline view of Zrythm, where the time signature automation is set to gradually change beats per bar from 4 to 3, and 3 to 4 at bars 3 and 4. The playhead is at what was supposedly the start of bar 3, which was changed to the start of the third beat in the bar 3. The entire timeline grid shows three beats for each bar.](https://images.lyuk98.com/430d1d6e-bafe-4b09-b575-0992e6ecf5ae.avif) |

Fortunately, the developers already [made a better implementation](https://mastodon.social/@zrythm/115535332863583356 "Zrythm DAW: \"Tempo & time signature automat…\" - Mastodon") for their upcoming major release.

After dealing with some limitations, I will stick to Ardour in the time being. However, when the [promised v2](https://forum.zrythm.org/t/some-updates-on-v1-release-and-v2-development "Some updates on v1 release (and v2 development) - News - Zrythm Forum") becomes available, I may give it another shot.
