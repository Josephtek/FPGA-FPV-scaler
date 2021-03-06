# FPGA-FPV-scaler

This is low-latency analog video to 720p HDMI video converter

It is designed for aggressive FPV flying, and so, optimized for minimal latency possible.


YCC -> RGB converter and HDMI output modules are borrowed from Mike Field HDMI processing project
https://github.com/hamsternz/Artix-7-HDMI-processing
I2C sender is borrowed (and slightly fixed) from Mike too

The circuit is composed from Ti TVP5150AM1 PAL/NTSC digital video decoder and Artix 7 FPGA. Some SDRAM is present on the development board, but not used now. 

Hardware used:

dev board
https://www.aliexpress.com/item/Xilinx-FPGA-Artix7-Artix-7-Development-Board-XC7A35T-Core-Board-with-SDRAM/32799960711.html

platform cable
https://www.aliexpress.com/item/Xilinx-Platform-USB-Download-Cable-Jtag-Programmer-for-FPGA-CPLD-XC2C256/32691266814.html

5150 dev module
https://www.aliexpress.com/item/TVP5150-Module-FPGA-SDRAM-PAL-Video-Decoding-Analog-AV-Input-Camera-VGA-Display/32456112655.html



Connections betwen FPGA chip and TVP5150 are pretty straightforward and can be derived from constraints file. I2C SDA and SCL lines are pulled-up at 5150 side to 3.3V with 2.2k resistors. Also, there is a small shottky diode in line with FPGA SCL out to mimic open-drain behavior. That's a last-minute hack, sorry :)

Also, this TVP5150 module lacks chip reset connection. You need to route a small wire to TVP's reset pin.

HDMI connection (DVI-D signalling strictly speaking) is made from HDMI cable cut in half. No big horrors here, but we need to remember that we are working witn picosecond timings (742.5MHz bit rate!). So all twisted pairs shall remain twisted and tidy to the end, no significant loops allowed. All shields shall be interconnected and grounded to module ground bus with shortest wire possible. Technically, passing HDMI thru 100mil pin headers is not the best idea, but its TMDS signalling seems to be robust enough to survive this.

Some HDMI devices (Glyph included) needs active +5V signal on HDMI cable to know the source is present. I used ~200 Ohm current-limiting resistor in series to avoid disasters.



There are two mode inputs defined on FPGA, mode_a and mode_b. Externally they are connected to two grounded switches and two 10K pull-up resistors to 3.3V.

Mode_b selects PAL or NTSC mode of scaler.
Mode_a can be used to momentary enforce syncing HDTV domain from input video if you do not want to wait about 20sec until digital PLL will lock for itself. Note that while that syncing by mode_a switch is enforced, output HDMI signal gets invalid (because pixel clock will run asynchronously to VSYNC) but will immediately be restored correct afterwards.

PDF schematics in docs section describes signal flow within the FPGA which is pretty straightforward. 

There are two clock domains. First is dotclock 27Mhz derived from 5150. It is used to fold video data to 16 bits and feed it into 32K x 16 FIFO buffer.

Second clock domain is generated from crystal on board and used to sync HDTV part. Pixel clock is 74.25MHz for 720p50 derived from PAL and 74.25/1.001 (damn it!) for 720p60/NTSC.

It is used to fetch data from FIFO and fill intermediate 720-pixel buffer which is used by deinterlacer and scaler. Also this clock drives 720p raster generator. Depending on current HDTV line number we are requesting new data from FIFO or do nothing, that way line just repeats itself doing line-tripling to convert NTSC sub-field lines to 720p or doing alternate doublings/triplings for PAL mode. (Bob deinterlacer combined with nearest-neighbour scaler, zero delay but some flicker, sorry)  

I2C module is pretty simple, it initializes TVP5150 with 3 bytes on startup.

Reset controller is just a reset timer.

A big mess of LUT blocks betweeen YCC -> RGB converter and HDMI out is vsync_visualizer module, meant for debugging timebase corrector DPLL. It shows a small vertical dash in the exact time of received  analog VSYNC against HDTV frame. If time is too early, dash is blue, if too late it is red, and if right on - green.

Based on this timing relation (generated in FIFO_reader module) we are bumping HDTV timing sligtly faster or slower to slowly lock output frame to input timing. Slow adjustment of HDTV clock is based on pshift trick possible on MMCM module of Xilinx 7 series devices. Single pshift shifts clock just for a few picoseconds. But if done continuously it will roll clock phase equivalent to about 500ppm frequency shift up/down. Pscontroller module controls this.

https://forums.xilinx.com/xlnx/board/crawl_message?board.id=7Series&message.id=15086

This project uses some generated intellectual property from Vivado. Everything is included in free license, but to compile this project you need to generate this IP yourself. I provided screenshots for all components (PLLs a and FIFO block). Actually only one component is tricky and needs to have all dividers entered as is (mult1001) others are relatively straightforward.

I have included compiled bitstream file and EEPROM config file, so if you want to try exactly the same dev board with pinout described in constraints xdc file, you can just download bitstream into FPGA.


Some cardinal sins of the existing design :)

1. Timings are violated many times in RTL (mainly because Vivado does not know well that there are two timing domains, needs better constraints file). Still no major glitches visible.
2. No auto PAL/NTSC detection. You need to set manual switch.
3. Moving PAL/NTSC switch violates FPGA MMCME module usage rules when it needs to be reset after clock change, and we do this even without switch debounce to add the insult. Still works most of the time for some reason :). Theoretically, we need to push FPGA reset button each time after change PAL/NTSC in this release (or add proper debounce and MMCME reset)

4. Locking onto input video takes about 20 seconds. 
5. When not locked (no time to sync) or when input video is heavily distorted because of reception interference, FIFO buffer gets overflown/underflown and video output gets garbled. HDMI timing stays rock solid though. May be fixed in future releases but not much problem because it shows only at startup or very momentary in the flight. Adding better FIFO filling and reading discipline will help. TI promised us that TVP5150AM1 BT.656 line length is always 720 pixels, but this seems not to be a case when input video is distorted.
6. Here is a big mess with 4 PLLE and one MMCME modules to generate all proper timings. One big source of this mess is NTSC's historical idiotic 1.001 frequency ratio which is not easy to syntheze in the existing FPGA PLLS. As Google loves to say "Sorry about this" ;) Very likely this circuit can be optimized a little bit with redundant PLLS thrown out, but even in current ugly state it does not seem to generate excessive clock jitter to cause any HDMI sync problems.


