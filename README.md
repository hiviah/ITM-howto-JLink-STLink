# ARM ITM trace PC sampling HOWTO for STLink and JLink/JTrace

This repository just contains magic incantations for GDB, openocd, JLink GDB server to enable ITM PC and exception sampling. ARM ITM trace is a feature of Cortex MCUs with CoreSight - it allows you to see what is going inside CPU and can act like profiler. Getting ITM to work with orbuculum.

This was tested on 3 boards, with STM32F407 and STM32F427 MCUs. If you have crystal oscillator speed different from 8 MHz, it might not work correctly. SWO speed should depend only on CPU core clock, however experimentally I found out it kind of doesn't or there is something weird going on with clock config. There will be note about this later.

First, clone orbuculum's devel branch, it contains some GDB macros for ITM settings and also `orbtop`:

    git checkout https://github.com/orbcode/orbuculum
    cd orbuculum
    git checkout Devel
    make

Notice file `Support/gdbtrace.init`, this contains lot of magic macros, we'll use it later.

## JLink/JTrace

Note for this you need original JLink/JTrace, the cheap chinese clones won't work. Power cycle JLink first, just to make sure.

First run JLink GDB server in one terminal, e.g.:

JLinkGDBServerCLExe -select USB -device Cortex-M4 -endian little -if SWD -speed auto -ir -LocalhostOnly

In GDB, connect to the server. This expects you have the `Support` dir from orbuculum in current directory. Magic incantations below. This is for CPU core clock of 168 MHz, look at the gdbtrace.init's comments to see what the arguments are. The monitor SWO EnableTargeT is part of JLink's GDB server, see JLink user guide.

The selected SWO speed below is 2000000 baud. You may have to reset target first via monitor reset.

The parameters below select PC sampling that shouldn't be too fast, otherwise you'd get a lot of ITM overflows. DWT POSTRESET setting is important in this.

    target extended-remote :2331
    source Support/gdbtrace.init
    monitor SWO EnableTarget 168000000 2000000 0xFF 0
    enableSTM32SWO 4
    prepareSWO 168000000 2000000 0 0
     
    dwtSamplePC 1
    dwtSyncTap 3
    dwtPostTap 1
    dwtPostInit 1
    dwtPostReset 15
    dwtCycEna 1
     
    ITMId 1
    ITMGTSFreq 3
    ITMTSPrescale 3
    ITMTXEna 1
    ITMSYNCEna 1
    ITMEna 1
     
    ITMTER 0 0xFFFFFFFF
    ITMTPR 0xFFFFFFFF
    continue

Now you should see some output if you do nc localhost 2332 to some file swo_data (port belongs to JLink GDB server and should pump out SWO data).

You can use pcsampl utility from these ITM tools. Let's try to parse the file you dumped from the port 2332, firmware.elf is the firmware running on your board:

    ./pcsampl -e firmware.elf swo_file 2>/dev/null

If the data are correct, you should see some meaningful result like:

        % FUNCTION
    10.77 *SLEEP*
    29.76 qstr_find_strn
    13.71 gc_collect_end
     6.97 mp_map_lookup
     6.00 gc_mark_subtree
     5.37 gc_alloc
     4.71 mp_execute_bytecode
     3.63 sha256_Transform
     1.80 mp_obj_get_type

You can also watch with `orbtop` which is part of orbuculum (look in `ofiles` directory). This will take data from JLink's SWO port 2332, show exceptions, max 15 lines. It's like top, it just shows time spent in functions instead:

    ./orbtop -E -e firmware.elf -v3 -c 15 -s localhost:2332

Sample output if all goes well:

     25.51%     2712 bn_multiply_reduce_step
     12.18%     1295 frexpf
     12.18%     1295 bn_multiply_long
      7.79%      828 qstr_find_strn
      4.65%      495 display_loader
      3.11%      331 gc_mark_subtree
      2.91%      310 mp_map_lookup
      2.26%      241 gc_alloc
      2.12%      226 bn_multiply_reduce
      2.11%      225 mp_execute_bytecode
      1.72%      183 sha256_Transform
      1.52%      162 bn_subtract
      1.43%      152 bn_add
      1.37%      146 bn_is_less
      1.27%      135 bn_rshift
    -----------------
     82.13%     8736 of 10627 Samples


     Ex |   Count  |  MaxD | TotalTicks  |  AveTicks  |  minTicks  |  maxTicks 
    ----+----------+-------+-------------+------------+------------+------------

    [---H] Interval = 1018mS / 0 (~0 Ticks/mS)

### Note on crystal oscillator speed different than 8 MHz

For some unknown reason the above GDB incantations will make CPU output 3 Mbaud SWO data instead of 2 Mbaud if oscillator speed onboard is 12 MHz (case of one tested board).

Just adjust the EnableTarget line to this, so that JLink expects 3 Mbaud data:

    monitor SWO EnableTarget 168000000 3000000 0xFF 0

Later I found out the code was expecting a 8 MHz oscillator on board and while the CPU ran at max speed (technically was set to "beyond max speed"), 
the prescaler dividers divided and this was one of the weird results.

## STLink

Success with STLink depends on STLink firmware version. Some work, some do not output SWO data, some randomly desync after you enable ITM trace.

One workaround I had success with was using USB-UART adapter and connect it directly to SWO pin of the CPU instead of passing it through STLink.

If you have the lucky version, run

    openocd -f interface/stlink-v2.cfg -f target/stm32f4x.cfg
    
and put followin into GDB. It enables SWO trace via STLink's protocol command, then enables PC sampling and exception sampling. We scale down the speed of sampling to avoid too many ITM overflows.

    target extended-remote :3333
    source Support/gdbtrace.init
    monitor tpiu config internal swodump.log uart off 168000000 2000000
    monitor mmw 0xE0001000 69632 0
    monitor mmw 0xE0001000 103 510
    dwtPostReset 15
    set *0xe0001000=*0xe0001000 | 0x200
    continue

It's a bit hairy mess, but should work (168 MHz core clock and 2 Mbaud SWO). If you are lucky and have the right STLink FW version, swodump.log file should appear in the directory where openocd was run. You can decode it as above with pcsampl.

If the swodump.log file is empty, you have the bad STLink FW version. You have to workaround via USB-UART adapter on SWO pin.

Set baudrate to 2 Mbaud, look at screen if it spews data. It's actually important to run screen as it seems to set some flags along with the stty. Might be issue with specific adapter.

    stty -F /dev/ttyUSB0 2000000
    screen /dev/ttyUSB0 2000000     # look if data is flowing, then kill screen
    stty -F /dev/ttyUSB0 2000000

Now you can either dump data via cat from `/dev/ttyUSB` and decode with pcsampl or you can run orbuculum with `orbtop` (each in separate terminal).

Both binaries below are in `ofiles` directory of orbuculum.

    ./orbuculum -p /dev/ttyUSB0 -a 2000000 -v2
    ./orbtop -E -e firmware.elf -v3 -p /dev/ttyUSB0 -a 2000000 -v2

If succesful, you'll see the the `orbtop` output like above.
Final remarks
Many roosters were sacrificed to make this work. You may experience different behavior if you are on unlucky FW version of JTAG/SWD adapters.

Logic analyzer and Pulseview/sigrok can help a lot to see whether you are getting data and whether it makes sense - screenshow below shows we are close, but there are framing errors with selected speed, so it's not precise:

![ITM over SWO where we almost matched the baudrate right](https://i.imgur.com/Pbnk3o4.png)

There are also some IDEs that could work with enabling this, like [STM32 Cube IDE](https://www.st.com/en/development-tools/stm32cubeide.html), however they have their own bugs. E.g. the Cube IDE definitely can't handle longer sampling because it tries to store everything into memory and in a very wasteful way at that. Even then it has problem with desync from adapter (this can be the FW issue) and depending on board will compute often wrong ITM settings, likely also the STLink FW issue. It is possible to generate [KCachegrind output with orbuculum](http://shadetail.com/blog/swo-instrumentation-building-the-orchestra/), but so far I haven't had much success with it. 

