
# Fusion 360 KINETIC-NC Post-Processor

A modified Autodesk Fusion 360 post-processor for the [HIGH-Z S-720/T](https://www.cnc-step.de/cnc-fraese-high-z-s-720t-kugelgewindetrieb-720x420mm) milling machine running the [KINETIC-NC](https://www.cnc-step.de/cnc-software/kinetic-nc-netzwerk-steuerungssoftware/) software, which both are from [CNC-Step](https://www.cnc-step.de). Autodesk Fusion post-processors are written in JavaScript and the documentation of the existing classes, functions, etc. is described in the [Autodesk CAM Post Processor Documentation](https://cam.autodesk.com/posts/reference/index.html).

The post processor is based on the classical format [RS-274D](https://en.wikipedia.org/wiki/G-code) also called [G-code](https://en.wikipedia.org/wiki/G-code). The original version of this RS-274D implementation for the FUSION 360 post-processor comes from [Benezan Electronics](http://www.benezan-electronics.de/index.html). You can download their post-processor [here](http://www.benezan-electronics.de/downloads/Autodesk_HSM_beamicon2.zip). For convenience I saved a copy in this repository.

The reason why the Benzean post-processor works for the KINETIC-NC software is that the KINETIC-NC software is based on the BEAMICON2 software of Benezan.

Thus I took above postprocessor and ran it successfully on FUSION 360 for my HIGH-Z S-720/T machine which is conrolled via the KINETIC-NC software. 

I did some modifications to the post-processor in order to make it more convenient for my typical operations.

Adding some commands to the post-processor is quite easy, as I needed to use only two already existing functions. The main function needed is [**writeln()**](https://cam.autodesk.com/posts/reference/classPostProcessor.html#aeb90bf455982d43746741f6dce58279c) which comes with the Autodesk JavaScript API and the second one is **writeComment()** which is a wrapper around **writeln()** that just adds brackets before and after the text (this is the comment format understood by KINETIC-NC).

## Example

Adding a section to [FUSION_360_KINETIC-NC_HIGH-Z_720T.cps](FUSION_360_KINETIC-NC_HIGH-Z_720T.cps) which is always written to the **\*.nc** file:

    writeln("");
    writeComment("Initial section");
    writeln("#100=350 (x-absolute machine coordinates)");
    writeln("#101=0   (y-absolute machine coordinates)");
    writeln("#102=0   (z-absolute machine coordinates, safety height)");
    writeln("");

## Explanation

    writeln("");
Adds an empty line because the string **""** is empty.

    writeComment("Initial section");

Write a comment line into the \*.nc file. The result in the file will look like this (the round brackets are added be the function *writeComment* to the text):

    (Initial section)

And KINETIC-NC interprets this as a comment.

    writeln("#100=350 (x-absolute machine coordinates)");
    writeln("#101=0   (y-absolute machine coordinates)");
    writeln("#102=0   (z-absolute machine coordinates, safety height)");

Above lines add variables **#100, #101, #102** to the \*.nc file. The values stored in these variables can be used later in the \*.nc file, e.g. for traversing like so:

    G53 G0 Z=#102 Y=#101 X=#100

In the past I added thoese lines always manually at the beginning of the \*.nc file in order to move safely to the workpiece without crashing into any clamps.  For my typical setups I then just need to adopt the initial x-coordinate. When I need to change tools the spindle drives back to machine zero and then comes back to the workpiece. So I added a safe path (by calling a subroutine which moves the spindle according to these variables) automatically before and after each tool change like:

    // go to safe position before doing tool change
    writeln('M98 P1234 (call subroutine 1234)');
    writeBlock("T" + toolFormat.format(tool.number), mFormat.format(6));
    writeln('M98 P1234 (call subroutine 1234)');

The line:

    writeBlock("T" + toolFormat.format(tool.number), mFormat.format(6));

writes following into the \*.nc file:

    N1000 T8 M6

Which means command number 1000 (for example), tool number 8 (for example) and **M6** is tool change. So I know when the post-processor writes this line a tool-change will happen. Thus I surround it by two lines which trigger a subroutine that performas the safe movement. KINETIC-NC allows a subroutine syntax:

 * M98 &rarr; call subroutine
 * P1234 &rarr; label of the subroutine (caller syntax)
 * O1234: &rarr; label of the subroutine (subroutine function starts here)
 * M99 &rarr; exit subroutine

The calls for adding the subroutine G-code within the post-processor look like this:

    // subroutines
    writeln("");
    writeln("");
    writeComment("Subroutine 1234");
    writeln("O1234:");
    writeln("G53 (machine coordinates)");
    writeln("G0 (go fast)");
    writeln("Z=#102 (safety height)");
    writeln("Y=#101");
    writeln("X=#100");
    writeln("G54 (workpiece coordinates)");
    writeln("M99 (End Subroutine 1234)");
    writeln("");

Which would render in the \*.nc file as (blank lines omitted):

    // subroutines
    (Subroutine 1234)
    O1234:
    G53 (machine coordinates)
    G0 (go fast)
    Z=#102 (safety height)
    Y=#101
    X=#100
    G54 (workpiece coordinates)
    M99 (End Subroutine 1234)
    
In the same way more functionality can be added to the post-processor or different *dialects* of the same post-processor could be created.
