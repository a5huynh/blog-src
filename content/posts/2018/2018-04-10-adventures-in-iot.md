+++
title = "Adventures in IoT: Dev Environment"

[taxonomies]
tags = ["iot", "hardware"]

[extra]
location = "San Francisco"
+++

As part of an Arduino hobby project, I wanted to set up a development
environment that steered clear of the Arduino IDE and gave me an
easy-to-use toolchain that fit in with my existing tools. While the Arduino
IDE is useful for simple sketches, once I moved into anything remotely
complex I yearned for the features provided by modern day text editors such
as Sublime Text or VS Code. This post documents my journey to the almost
perfect (for me) dev environment for Arduino and other micro-controllers.

<!-- more -->

## Requirements

Before I started down the path of customizing everything I had a couple
constraints:

* Standard text editor, with standard text editing features.
* Syntax highlighting, customized to my liking.
* Code completion/IntelliSense.
* Command line compilation and upload.
* And if possible, support for other devices and good documentation.


## VS Code Arduino Extension

My daily text editor for a while has been [Visual Studio Code][vscode], an
electron based editor that wooed me over from native editors like Sublime
Text through a rich ecosystem of extensions and just really great sane
defaults. Hoping I’d still be able to use VS Code as part of my Arduino dev
process, I was able to find the [VS Code Arduino extension][vscode-ext]
developed by none other than Microsoft.

The extension is currently in preview, with a couple issues that may or may
not affect your workflow. It interfaces with the Arduino IDE to bring over
some handy functionality:

* Board & library manager.
* Serial monitor.
* Integrated Arduino debugging (through [OpenOCD][openocd]).

[openocd]: http://openocd.org
[vscode]: https://code.visualstudio.com
[vscode-ext]: https://github.com/Microsoft/vscode-arduino


### Getting IntelliSense to work

Fortunately and unfortunately the extension only get us 80% of the way
there. I ran into errors where Arduino symbols like `pinMode` or `LOW/HIGH`
were undefined and which seemed to be a [common issue][ext-issue] that
other Arduino hobbyists encountered.

The fix is simple, we need to set up our include paths so that VS Code
C/C++ IntelliSense parser knows where to look. Since part of the issue is
the inability to recursively walk through the include folders, we'll need
to explicitly add each folder. This can be accomplished by editing the
`c_cpp_properties.json` file to include the following paths:

> Note: The `c_cpp_properties.json` file can be accessed via the command
> palette: `⌘⬆P -> C/Cpp: Edit Configurations`.

``` javascript,linenos
// note to save some space,
// $PACKAGES = $HOME/Library/Arduino15/packages/arduino
"includePath": [
    "$PACKAGES/hardware/avr/1.6.21/cores/arduino",
    "$PACKAGES/hardware/avr/1.6.21/libraries",
    "$PACKAGES/hardware/avr/1.6.21/variants/standard",
    "$PACKAGES/tools/avr-gcc/4.9.2-atmel3.5.4-arduino2/avr/include",
    "$PACKAGES/tools/avr-gcc/4.9.2-atmel3.5.4-arduino2/lib/gcc/avr/4.9.2/include",
    "${workspaceFolder}"
]
```

The versions in the paths will change depending on how often you update
your Arduino IDE libraries, but should cover all the standard Arduino
includes. If you need additional libraries, you should be able to find the
paths for those roughly around the same area. Custom libraries, such as
ones you’ve written yourself, should ideally be in a `lib` folder under
the same folder as your main sketch file. For example if you have a
`sensor` library it’d look like below. The importance of the `lib` folder
structure will come into play in the toolchain section.

```
project/
├── lib/
│   └── sensor/
│       ├── sensor.cpp
│       └── sensor.h
├── Makefile
└── main.ino
```

Lastly and most importantly, we need to add `#include <Arduino.h>` at the
top of our main sketch file so that the VS Code IntelliSense to start
working its magic.

[ext-issue]: https://github.com/Microsoft/vscode-arduino/issues/438


## Arduino Toolchain

With the text editor now working within the requirements I set earlier, I
looked towards compiling and uploading an Arduino sketch file using nothing
but the command line. After a little searching I stumbled upon
[Arduino Makefile][makefile], a fantastic open source project with a very
configurable workflow for sketch compilation and upload.

[makefile]: https://github.com/sudar/Arduino-Makefile


### Issues and Fixes

Following the example Makefiles, getting up and running with a single
sketch file was easy. As more files were added to the project there
were a couple non-obvious changes I needed to make to my project structure
and the Makefile to support user-defined libraries.

First, user-defined libraries should have a folder structure that follows a
familiar format:

```
lib/
└── <library>/
│   ├── <library>.cpp
│   └── <library>.h
```

In the Makefile, we’ll need to define two variables which define the
libraries that should be included in the compilation process and the path
to the user-defined libraries.

``` bash
BOARD_TAG = uno
ARDUINO_LIBS = <library name> <library name two>
USER_LIB_PATH = $(PWD)/lib

# The following path may be different for you. This path
# is for OSX users who installed Arduino Makefile via Homebrew
include /usr/local/opt/arduino-mk/Arduino.mk
```

Now we should be able to run `make`  to compile the project and `make
upload` to upload the sketch to the connected Arduino board. The Arduino
Makefile will attempt to auto-detect variables based on your current OS and
settings that you have setup already in the Arduino IDE. If you need to
override any of variables, such as setting the `BOARD_TAG` to the board
you’re using, simply put the user-defined variable before including the
Arduino Makefile.

For easier project management, the best practice is to include Arduino
Makefile as a git submodule in your project repo and set the include path
to the submodule.


## Serial Monitor/Debugging

Once we’re able to upload a sketch to our Arduino, the last step would be
to monitor and interactively input data into the serial port connection.
This can be for any logs we’re outputting, setting up configurations, etc.
Luckily, if you’re using any of the tools above there are plenty of ways to
accomplish this.

If you’re using VS Code, the Arduino extension has commands to open a
serial connection, set the baud rate, and select the serial port.
Unfortunately, the Arduino extension only allows you to read from the
serial port.

Arduino Makefile has a built-in command, `make monitor` that connects to
the Arduino with the correct incantations through the `screen` command.

While this works just fine, I’ve always enjoyed using the feature-ful
`minicom` to connect to serial ports. It takes a little configuration but
the end result is a more powerful way to interact with the Arduino serial
connection.

Minicom is available on OSX via homebrew:

```
homebrew install minicom
```

To configure minicom, start it up with `minicom -s` to prevent it from
immediately attempting to connect to any serial ports.  From here you can
set the serial device to point to the usb serial port
(`/dev/cu.usbserial-XXXX`) and set `Hardware Flow Control` to `Off` so that
you can send data through the serial port. Save these settings as the default
and every time you run `minicom` it’ll start up with those settings and
immediately connect to the serial port.


## Conclusion

And that's it! I've so far used the above configuration for about a month
so far with no major problems. Next up on this adventure, I dive deeper
into a project which involves sensors, pumps, and even WiFi communication
for a more automated home.
