# LiRover Journal

Here's basically everything that's happened so far with the rover:

* Actually laid out the custom PCB for real in KiCad instead of just sketching it on paper. It's built around a Pico W for the low level motor/sensor stuff, with the Pi Zero's 40 pin header and an SD card slot right next to it.
* Added a proper connector for the LiDAR module so it's not just dangling wires anymore.
* Power section now has a fuse and some caps (220uF/10uF bulk + 100nF decoupling) so hopefully we don't fry anything the first time we plug it in.
* Swapped from soldered wires to screw terminals for the motor/battery connections since we're going to be taking this apart constantly while we test stuff.
* Threw in a spare 1x4 header we don't need yet but will probably want later.
* Added keepout zones over the Pico for the antenna and the USB port so copper doesn't end up somewhere it shouldn't be. These are rough estimates for now, still need to double check against the actual Pico W footprint before we send anything off to get made.
* Four M3 mounting holes in the corners so the board can actually bolt down to the chassis instead of floating around in there.
* Designed and printed a little switch housing, box with a lid and a button poking through the top, so we can hit the power switch without exposing any of the wiring underneath.

Next up is wiring in the IMU and encoders now that the connectors are pretty much locked in, and make sure the CAD and PCB are fully done and polished.
