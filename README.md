<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [cli-visualizer](#cli-visualizer)
  - [Installing Pre-requisites](#installing-pre-requisites)
    - [256 Colors](#256-colors)
    - [Ubuntu](#ubuntu)
    - [Arch Linux](#arch-linux)
    - [Mac OS X](#mac-os-x)
  - [Installing](#installing)
    - [Arch Linux](#arch-linux-1)
  - [Setup](#setup)
    - [MPD Setup](#mpd-setup)
    - [ALSA Setup](#alsa-setup)
      - [ALSA with dmix](#alsa-with-dmix)
    - [Pulse Audio Setup (Easy)](#pulse-audio-setup-easy)
  - [Usage](#usage)
  - [Configuration](#configuration)
    - [Reloading Config](#reloading-config)
    - [Colors](#colors)
      - [RGB colors](#rgb-colors)
      - [Color indexes](#color-indexes)
      - [Color names](#color-names)
    - [Spectrum](#spectrum)
      - [Smoothing](#smoothing)
        - [Sgs Smoothing](#sgs-smoothing)
        - [MonsterCat Smoothing](#monstercat-smoothing)
        - [No Smoothing](#no-smoothing)
      - [Falloff](#falloff)
      - [Spectrum Appearance](#spectrum-appearance)
    - [Full configuration example](#full-configuration-example)
  - [Trouble Shooting](#trouble-shooting)
    - [Mac OSX](#mac-osx)
        - [vis hangs with no output](#vis-hangs-with-no-output)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

![travis-ci](https://travis-ci.org/dpayne/cli-visualizer.svg?branch=master)

![Coverity Scan Build Status](https://scan.coverity.com/projects/7519/badge.svg)

# cli-visualizer

Command line visualizer. Supports mpd, with experimental support for alsa and pulseaudio.

This project was heavily inspired by [C.A.V.A](https://github.com/karlstav/cava), [ncmpcpp](http://rybczak.net/ncmpcpp/), and [rainbow](https://github.com/sickill/rainbow)

![spectrum_stereo](http://i.imgur.com/BGyfYiv.gif "Spectrum Stereo")

![spectrum_mono](http://i.imgur.com/Zo5lByM.gif "Spectrum Mono")

![ellipse](http://i.imgur.com/WoQlObi.gif "Ellipse")

![lorenz](http://i.imgur.com/9QJjnDI.gif "Lorenz")


## Installing Pre-requisites

`fftw` and `ncursesw` libraries are required to build. Note that ncurses with wide character support is needed.

A C++ compiler that supports C++14 is also needed. On arch linux, the latest g++ or clang++ will work.

### 256 Colors

In order to show the colors, you need a terminal with 256 color support.`rxvt-unicode` out of the box.

For xterm, the default `$TERM` setting needs to be changed to `xterm-256color`. To change this run

    export TERM=xterm-256color

### Ubuntu


	sudo apt-get install libfftw3-dev libncursesw5-dev

For pulseaudio support, the pulseaudio library also needs to be installed

	sudo apt-get install libpulse-dev

Older versions of Ubuntu also need newer a newer gcc compiler. Note, while this should be safe, it will upgrade some base libc libraries that might break your system.

	sudo add-apt-repository ppa:ubuntu-toolchain-r/test
	sudo apt-get update
	sudo apt-get install gcc-4.9 g++-4.9

### Arch Linux

In arch, the ncursesw is bundled with the ncurses package.

    sudo pacman -S ncurses fftw

### Mac OS X

Mac os x has a version of ncurses builtin, but a newer version is required.

	brew install fftw
	brew tap homebrew/dupes
	brew install ncurses

## Installing

After the pre-requisites have been installed, run the install script.

    ./install.sh

The configuration file is installed under "~/.config/vis/config".

Older version of Ubuntu need to compile with the newer g++ compiler.

	export CXX=g++-4.9
	./install.sh

### Arch Linux

The Arch Linux install is much simpler since an AUR package exist for cli-visualizer.

	yaourt -S cli-visualizer

You will have to copy config and colors manually.

## Setup

At least one of the following needs to be configured (and linked in the config): mpd, alsa and pulseaudio

### MPD Setup

The visualizer needs to use mpd's fifo output file to read in the audio stream. To do this, add the following lines to your mpd config file.


	audio_output {
    	type                    "fifo"
    	name                    "my_fifo"
    	path                    "/tmp/mpd.fifo"
    	format                  "44100:16:2"
	}

If you have any sync issues with the audio where the visualizer either get ahead or behind the music, try lowering the audio buffer in your mpd config.

	audio_output {
        ...
        #this sets the buffer time to 50,000 microseconds
        buffer_time     "50000"
	}

### ALSA Setup

WARNING: alsa support is very very experimental. It is very possible that it will be completely broken on various systems and could cause weird/bad behaviour.

Similar to the MPD setup, the visualizer needs to use a fifo output file to read in the alsa audio stream. To do this, add the following lines to your alsa config file, usually at `/etc/asound.conf`. If the file does not exist create it under `/etc/asound.conf`. 

    pcm.!default {
        type file               # File PCM
        slave.pcm "hw:0,0"      # This should match the playback device at /proc/asound/devices
        file "|safe_fifo /tmp/audio" #safe_fifo will be in the cli-visualizer/bin/safe_fifo directory
        format raw              # File format ("raw" or "wav")
        perm 0666               # Output file permission (octal, def. 0600)
    }

Next change the visualizer config `~/.config/vis/config` to point to the new fifo file

    mpd.fifo.path=/tmp/audio

A normal fifo file can not be used since otherwise no sound would work unless the fifo file is being read. This effectively means that no sound would be played if the visualizer is not running. To get around this issue a helper program `safe_fifo` is used. The `safe_fifo` program is essentially a non-blocking fifo file, it takes `stdin` and writes it to a fifo files given as the first parameter. If the fifo buffer is full, it clears the buffer and writes again.

Note that alsa support is still very experimental. There are a couple of caveats with this approach. Firstly `safe_fifo` must in a location alsa can find it. If you are building from source it us under `cli-visualizer/bin/safe_fifo`. Secondly `slave.pcm` must match whatever your alsa playback device is.

#### ALSA with dmix

On sound cards that do not support hardware level mixing, alsa uses dmix to allow playback from multiple applications at once. In order to make this work with the visualizer a dmixer plugin needs to be defined in `/etc/asound.conf` and the visualizer pcm set to use `dmixer` instead of the hardware device directly. This configuration might will change slightly depending on the system. 

This is an example `asound.conf` for Intel HD Audio.

    pcm.dmixer { 
        type dmix 
        ipc_key 1024
        ipc_key_add_uid false
        ipc_perm 0666            # mixing for all users
        slave { 
            pcm "hw:0,0" 
            period_time 0 
            period_size 1024 
            buffer_size 8192
            rate 44100
        }
        bindings { 
            0 0 
            1 1 
        } 
    } 

    pcm.dsp0 { 
        type plug 
        slave.pcm "dmixer" 
    } 

    pcm.!default { 
        type plug 
        slave.pcm "dmixer" 
    } 

    pcm.default { 
       type plug 
       slave.pcm "dmixer" 
    } 

    ctl.mixer0 { 
        type hw 
        card 0 
    }

    pcm.!default {
        type file               # File PCM
        slave.pcm "dmixer"      # Use the dmixer plugin as the slave pcm
        file "|safe_fifo /tmp/audio"
        format raw              # File format ("raw" or "wav")
        perm 0666               # Output file permission (octal, def. 0600)
    }

### Pulse Audio Setup (Easy)

Pulse audio should be the easiest to setup out of all the options. In order for this to work pulseaudio must be installed and vis must be built with pulseaudio enabled. To build with pulseaudio support run `make ENABLE_PULSE=1`.

To enable pulse audio in the vis config set audio sources to pulse with

    audio.sources=pulse

If this does not work, then vis has most likely not guess the correct sink to use. Try switching the pulseaudio source vis uses. A list can be found by running the command `pacmd list-sinks  | grep -e 'name:'  -e 'index'`. The correct pulseaudio source can then be set with

    audio.pulse.source=0

## Usage

Start with

	vis

###Controls

| Key | Description |
| --- | ----------- |
| <kbd>space</kbd> | switch visualizers |
| <kbd>q</kbd> or <kbd>CTRL-C</kbd>| Quit |
| <kbd>r</kbd>| Reload config |

## Configuration

### Reloading Config

The config can be reload while `vis` is running by either pressing the `r` key or by sending the `USR1` signal to `vis`. Sending the `USR1` signal can be done with `killall -USR1 vis`, this is useful if you want to dynamically reload colors from a script.

### Colors

The display colors and their order can be changed by switching the color scheme in the config under `colors.scheme`.
The color scheme must be defined at `~/.config/vis/colors/color_scheme` directory. There are three different ways to specific a color, by name, by hex number, and by index.

vis does not override or change any of the terminal colors. All colors will be influenced by whatever terminal settings are set by the users terminal. Usually these colors are specified in `.Xdefaults`. There are three different ways to specific a color, by name, by hex number, and by index.

#### RGB colors

The color scheme can be defined in two ways with RGB values or by index. The RGB values are defined as a hex color code, they are useful for creating a 256 bit color scheme.
The displayed color will not be the true color, but instead an approximation of the color based on 256-bit terminal colors.

RGB color scheme example

    #4040ff
    #2c56fc
    #2a59fc
    #1180ed
    #04a6d5
    #02abd1
    #03d2aa
    #04d6a5
    #12ee7f
    #2bfc58
    #2dfc55
    #56fc2d

#### Color indexes

The second way to define a color scheme is by the color index. Specifically this is the exact color index used by ncurses.
Color indexes are useful for creating a visualizer that matches the terminal's set color scheme.

Color index color scheme example

    4
    12
    6
    14
    2
    10
    11
    3
    5
    1
    13
    9
    7
    15
    0


![basic_colors](http://i.imgur.com/TUrQU01l.gif "Basic Colors")

All the basic 16 terminal colors from 0-16 in-order. The spectrum colors can be ordered in any you want, this example was done in order to show all colors.

<br><br>


#### Color names

The third way to define a color scheme is by the color name. Only 8 color names are supported: black, blue, cyan, green, yellow, red, magenta, white.
Note that these 8 colors are the basic terminal colors and are often set by a terminal's color scheme. This means that the color name might not match the color shown since the terminal theme might change it.
For example, dark themes often set `white` to something dark since `white` is usually the default color for the terminal background.

![blue](http://i.imgur.com/vn1u9tVl.gif "Blue")

A color scheme with only one color `blue`.

<br><br>


### Spectrum

The spectrum visualizer allows for many different configuration options.

#### Smoothing

There are three different smoothing modes, monstercat, sgs, none.

    #Available smoothing options are monstercat, sgs, none.
    visualizer.spectrum.smoothing.mode=sgs

##### Sgs Smoothing

SGS smoothing [Savitzky-Golay filter](https://en.wikipedia.org/wiki/Savitzky%E2%80%93Golay_filter). There are a couple of options for sgs smoothing.

The first option is controlling the number of smoothing passes done when rendering the bars. Increasing this number with make the spectrum smoother. This is set with `visualizer.sgs.smoothing.passes`.

The second option is the number of neighbors to look at when smoothing. This is set with `visualizer.sgs.smoothing.points`. This should always be an odd number.
A larger number will generally increase the smoothing since more neighbors will be looked at.

![sgs_smoothing](http://i.imgur.com/fuxuNQTl.gif "Sgs Smoothing")

Default sgs smoothing.

<br><br>


![sgs_smoothing](http://i.imgur.com/ptF0UhBl.gif "Sgs Smoothing High Pass")

Sgs smoothing with number of passes set to `5`.

##### MonsterCat Smoothing

Monster cat smoothing is inspired by the monster cat youtube channel (https://www.youtube.com/user/MonstercatMedia). To control the amount of smoothing for montercat use `visualizer.monstercat.smoothing.factor`. The default smoothing factor for monstercat is `1.5`. Increase the smoothing factor by a lot could hurt performance.

![monstercat_smoothing](http://i.imgur.com/1wmgrjel.gif "MonsterCat")


##### No Smoothing

Smoothing can be completely turned off by setting the smoothing option to `none`.

    visualizer.spectrum.smoothing.mode=none

Spectrum with smoothing off

![none_smoothing](http://i.imgur.com/8vLS91kl.gif "No Smoothing")


#### Falloff

This configures the falloff effect on the spectrum visualizer. This effect creates a slow fall in bar height. Available falloff options are `fill,top,none`. The default falloff option is `fill`.

    visualizer.spectrum.falloff.mode=fill

With the `top` setting the falloff effect is only applied to the top character in the bar. This creates a gap between the main bar and the top falloff character.

![top_falloff](http://i.imgur.com/F64NBWY.gif "Top falloff")

top falloff effect with the spectrum character set to `#`.
<br><br>

The `fill` option leaves no gaps between the very top character and the rest of the bar.


![fill_falloff](http://i.imgur.com/RUOUkKQ.gif "Fill falloff")

fill falloff effect with the spectrum character set to `#`.

<br><br>

The `none` option removing the falloff effect entirely..

![none_falloff](http://i.imgur.com/DXPQdp2.gif "No Falloff")

Falloff effect removed with the spectrum character set to `#`.

#### Spectrum Appearance

Certain aspects of the spectrum appearance can be controlled through the config.

The bar width can be controlled to make it wider or narrower. The default is `2`.

    visualizer.spectrum.bar.width=2

![bar_width](http://i.imgur.com/LT5wRGVl.gif "Wide bar width")

Bar width set to `5`.

<br><br>

The spacing between bars can also be controlled to make it wider or narrower. The default is `1`.

    visualizer.spectrum.bar.spacing=1

![bar_spacing](http://i.imgur.com/VO8oHZBl.gif "No bar spacing")

Bar spacing set to `0`.

<br><br>

The margin widths of the spectrum visualizer can also be controlled. The margins are set in percent of total screen for spectrum visualizer. All margin percentages default to `0.0`.

    visualizer.spectrum.top.margin=0.0
    visualizer.spectrum.bottom.margin=0.0
    visualizer.spectrum.right.margin=0.0
    visualizer.spectrum.left.margin=0.0

![spectrum_margins](http://i.imgur.com/t1t3cdjl.gif "Spectrum Margins")

Spectrum with the top margin set to `0.30` and the left and right margins set to `0.10`.

<br><br>

Normally the bars are ordered so that the lowest frequencies are handled by the left most bars and the highest frequencies by the right most bars.
The reversed option gives the option to reverse this so that the highest frequencies come first.

    visualizer.spectrum.reversed=true

![spectrum_reversed](http://i.imgur.com/EDhaRDrl.gif "Reversed")

Spectrum with reverse set to `true`.

### Full configuration example

    #Refresh rate of the visualizers. A really high refresh rate may cause screen tearing. Default is 20.
    visualizer.fps=20

    #Defaults to "/tmp/mpd.fifo"
    mpd.fifo.path

    #If set to false the visualizers will use mono mode instead of stereo. Some visualizers will
    #behave differently when mono is enabled. For example, spectrum show two sets of bars.
    audio.stereo.enabled=false

    #Specifies how often the visualizer will change in seconds. 0 means do not rotate. Default is 0.
    visualizer.rotation.secs=10

    #Configures the samples rate and the cutoff frequencies.
    audio.sampling.frequency=44100
    audio.low.cutoff.frequency=22050
    audio.high.cutoff.frequency=30


    #Configures the visualizers and the order they are in. Available visualizers are spectrum,lorenz,ellipse.
    #Defaults to spectrum,ellipse,lorenz
    visualizers=spectrum,ellipse,lorenz


    #Configures what character the spectrum visualizer will use. Specifying a space (e.g " ") means the
    #background will be colored instead of the character. Defaults to " ".
    visualizer.spectrum.character=#

    #Spectrum bar width. Defaults to 1.
    visualizer.spectrum.bar.width=2

    #The amount of space between each bar in the spectrum visualizer. Defaults to 1. It's possible to set this to
    #zero to have no space between bars
    visualizer.spectrum.bar.spacing=1

    #Available smoothing options are monstercat, sgs, none. Defaults to sgs.
    visualizer.spectrum.smoothing.mode=monstercat

    #This configures the falloff effect on the spectrum visualizer. Available falloff options are fill,top,none.
    #Defaults to "fill"
    visualizer.spectrum.falloff.mode=fill

    #Configures how fast the falloff character falls. This is an exponential falloff so values usually look
    #best 0.9+ and small changes in this value can have a large effect. Defaults to 0.95
    visualizer.spectrum.falloff.weight=0.95

    #Margins in percent of total screen for spectrum visualizer. All margins default to 0
    visualizer.spectrum.top.margin=0.0
    visualizer.spectrum.bottom.margin=0.0
    visualizer.spectrum.right.margin=0.0
    visualizer.spectrum.left.margin=0.0

    #Reverses the direction of the spectrum so that high freqs are first and low freqs last. Defaults to false.
    visualizer.spectrum.reversed=false

    #Sets the audio sources to use. Currently available ones are "mpd" and "alsa"Sets the audio sources to use.
    #Currently available ones are "mpd", "pulse", and "alsa". Defaults to "mpd".
    audio.sources=pulse

    ##vis tries to find the correct pulseaudio sink, however this will not work on all systems.
    #If pulse audio is not working with vis try switching the audio source. A list can be found by running the
    #command pacmd list-sinks  | grep -e 'name:'  -e 'index'
    audio.pulse.source=0

    #This configures the sgs smoothing effect on the spectrum visualizer. More points spreads out the smoothing
    #effect and increasing passes runs the smoother multiple times on reach run. Defaults are points=3 and passes=2.
    visualizer.sgs.smoothing.points=3
    visualizer.sgs.smoothing.passes=2


    #Configures what character the ellipse visualizer will use. Specifying a space (e.g " ") means the
    #background will be colored instead of the character. Defaults to " ".
    visualizer.ellipse.character=#

    #The radius of each color ring in the ellipse visualizer. Defaults to 2.
    visualizer.ellipse.radius=2

    #Specifies the color scheme. The color scheme must be in ~/.config/vis/colors/ directory. Default is "colors"
    colors.scheme=rainbow

## Trouble Shooting

### Mac OSX

#### vis hangs with no output

It is possible there is a naming conflict with an existing BSD tool `vis`. Try running `vis -h`. If the output looks something like

    mac ❯❯❯ vis -h
    vis: illegal option -- h
    usage: vis [-cbflnostw] [-F foldwidth] [file ...]

then there is a naming conflict with cli-visualizer. To fix this issue, put `/usr/local/bin` before `/usr/bin/` in `$PATH`.
