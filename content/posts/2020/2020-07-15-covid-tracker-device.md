+++
title = "COVID-19 Stats Tracker"

[taxonomies]
tags = [
    "esp32",
    "hardware",
    "iot",
]

[extra]
location = "San Francisco"
+++

One of things I wanted to accomplish when I began learning more about
electronics was to build a small device from the ground up. This would
include sourcing all parts, writing the necessary code, and designing and
3D printing an enclosure for the device. Given the state the world at this
time, as my first foray into electronics design, I decided to make a little
device to keep track of [US COVID-19 stats][covid-api].

[covid-api]: https://covidtracking.com/api/us

<!-- more -->

## Bill of Materials

* **1** × [Wemos LOLIN D32][wemos-link]
* **1** × [MSP2402 SPI LCD Screen][lcd-link]
* **1** × Protoboard
* **3** × [JST connectors][jst-connectors]
* **2** × [Pin headers][pin-headers]

Starting off small, I used a micro-controller that came with everything and
the kitchen sink. The [Wemos LOLIN D32][wemos-link] is a fantastic little
piece of electronics that cost around $5. It's a dev board based around the
[ESP32-WROOM-32][esp32-wiki] module and is compatible with the Arduino
platform. and comes with WiFi and Bluetooth out of the box. Perhaps a
little overkill for what I'm building, but it allowed me to get up and
running very quickly.

The MSP2402 screen is an 2.4 inch TFT LCD Screen with a resolution of
320x240 pixels. I chose a larger screen so that I can could make something
large enough for "at-a-glance" stats that I could put on a shelf somewhere.

All major pieces were connected together on a protoboard using JST connectors and
pin headers so that future revisions can reuse the same components.

[esp32-datasheet]: https://www.espressif.com/sites/default/files/documentation/esp32-wroom-32_datasheet_en.pdf
[esp32-wiki]: https://en.wikipedia.org/wiki/ESP32
[lcd-link]: http://www.lcdwiki.com/2.4inch_SPI_Module_ILI9341_SKU:MSP2402
[wemos-link]: https://docs.wemos.cc/en/latest/d32/d32.html
[jst-connectors]: https://en.wikipedia.org/wiki/JST_connector
[pin-headers]: https://en.wikipedia.org/wiki/Pin_header

## Writing the Code

The code for the project is available on [GitHub](https://github.com/a5huynh/embedded/tree/master/esp32-display).

The project utilizes a couple useful utilities to make development a lot
easier. First and foremost is the [PlatformIO IDE](https://platformio.org/)
plugin for VS Code. This brings some modern niceties that aren't available
with the standard Arduino IDE such as dependency version management, a
command line based build system, and the ability to use the code editor of
your choice.

For the implementation itself, I used:

* [Arduino HTTPClient library][arduino-http] to make requests to the COVID-19 stats API
* [ArduinoJson][arduino-json] to parse the API results
* [TFT_eSPI][tft-espi] to display the results

The basic loop function would look like something below:
``` c,linenos
bool updateScreen = false;

void loop() {
    if (wifi.run() == WL_CONNECTED) {
        unsigned long currentTime = millis();
        if (currentTime - LAST_UPDATED_TIME >= UPDATE_TIME_MS) {
            LAST_UPDATED_TIME = currentTime;
            update_tracker()
        }
    }

    if (updateScreen) {
        render_stats();
    }
}
```

We want to make sure we have an internet connection (`wifi.run()`) and if everything
looks good, we can go ahead and make a request to the API. If anything has changed
in the stats, we'll set a flag to update the screen. Once that request is done,
if the screen update flag is set, we can run through the code to re-render the
contents of the screen.

All in all, the code was very straightforward and pretty similar to how you would
build a similar stats tracker in any other language/platform.

[arduino-http]: https://github.com/espressif/arduino-esp32/tree/master/libraries/HTTPClient
[arduino-json]: https://github.com/bblanchon/ArduinoJson
[tft-espi]: https://github.com/Bodmer/TFT_eSPI


## Wiring Schematic

Now for the hardware.

To keep track of all the different wires, I used
[kicad](https://kicad-pcb.org/) to build out a schematic based on the
breadboard prototype I had built. As I get better at kicad, my hope is
eventually turn some of these schematics into actual PCBs that can be
printed, but for the time being, it's a handy way to visualize the
connections.

![covid](/img/2020/covid-tracker-schematic.png)

For this first build, I left out the button which allows you to jump to different
screens, but the code and schematic still handles this.

Below are some photos of the final soldered result:

* Photo #1: Overview of the entire project.
* Photo #2: Closeup of the soldering for the main board.
* Photo #3: Closeup of the LCD setup.

{{ gallery(images=[
    "/img/2020/covid-tracker/overview.jpg",
    "/img/2020/covid-tracker/soldering-work.jpg",
    "/img/2020/covid-tracker/lcd.jpg"
]) }}


## Designing & Printing an Enclosure

For the enclosure design, I settled on using [Shapr3D][shapr3d-link] for
the iPad. It is a fantastic application that has everything I need as a
hobbyist to get things done. Since the enclosure is nothing more than a
simple rectangular box, diving into more complex tools like Fusion 360 or
AutoCAD just didn't seem like a lot of fun.

This was my first enclosure that was designed from scratch, and is comprised of two
large halves that are screwed together. The files are available [here for download][v1-link].

I used my MonoPrice Maker Select v2 to print out the enclosure and test prototypes
before finally putting everything together.


[shapr3d-link]: https://www.shapr3d.com/
[v1-link]: /files/2020/covid-tracker-box-v1.zip

## Finished Result!

Here's a video fo the COVID tracker booting up and loading the data

<iframe width="560" height="315" src="https://www.youtube.com/embed/h8oSt7ZR6MU" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Some things I would like to work on for v2 of this enclosure:

* New enclosure with better assembly and holes for buttons. The two halves was a
  naive design decision due to inexperience and made assmebly more difficult
  than necessary.
* Adding additional screens that can be toggled via buttons.
