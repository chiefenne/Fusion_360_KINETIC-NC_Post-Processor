
# Fusion 360 KINETIC-NC Post-Processor



A modified Autodesk Fusion 360 post-processor for the [HIGH-Z S-720/T](https://www.cnc-step.de/cnc-fraese-high-z-s-720t-kugelgewindetrieb-720x420mm) milling machine running the [KINETIC-NC](https://www.cnc-step.de/cnc-software/kinetic-nc-netzwerk-steuerungssoftware/) software, which both are from CNC-Step (https://www.cnc-step.de).

Autodesk Fusion post-processors are written in JavaScript and the documentation of the existing classes, functions, etc., is described in the [Autodesk CAM Post Processor Documentation](https://cam.autodesk.com/posts/reference/index.html).

I made some modifications to the post-processor provided by CNC-Step, in order to make it more convenient for my typical operations. Be aware, that the modifications are tested only the KINETIC-NC software and on my personal machine. KINETIC-NC supports an additional set of specific commands which are not part of the RS-274D standard. KINETIC-NC supports also a macro language, which can be used to automate recurring tasks.

Adding/modifying the functionality of the post-processor is convenient when using the [Visual Studio Code](https://code.visualstudio.com/) editor and the [Autodesk Fusion Post Processor Utility](https://marketplace.visualstudio.com/items?itemName=Autodesk.hsm-post-processor) extension. 

Mainly two functions from the post API are employed. The main function needed is **writeln()** which comes with the Fusion JavaScript API. The second one is **writeComment()** which is a wrapper around **writeln()** that just adds brackets before and after the text (this is also the comment format understood by KINETIC-NC).

A third function written by myself is **safeStartPositionChiefenne()**. This is used to go to a "safe" start position. In my setups I have most of the time a situation where my workpiece coordinate system is G54. Further I clamp the stock so that I can safely approach the G54 origin. First I traverse at y=0 along the x-axis (to G54 X0), then I move along the y-axis (to G54 Y0). The spindel is typically left at z=0.

## Features

 * Initial section
   - Auto-read x-position of actual G54
   - Request user to confirm or enter alternative offset
 * Safe tool position
   - Before and after tool change
   - At program end
   - Implemented via subroutine call *safeStartPositionChiefenne*
 * Jump labels between sections (e.g., 2D adaptive clearing, pockets, etc.)
   - Allows to execute individual sections separately
 * Repeat / Next within each section
   - Allows easily to redo sections multiple times
 * Remove entries not understood by KINETIC-NC
   - Remove **%** from last line

## Example code snippet

Adding a section to [FUSION_360_KINETIC-NC_HIGH-Z_720T.cps](FUSION_360_KINETIC-NC_HIGH-Z_720T.cps) which will always be written to the **\*.nc** file when using this post-processor:

```JavaScript
writeln("");
writeComment("Safe path to workpiece origin - derived automatically from G54 x-coordinate");
writeComment("So we need to switch to G54 before running the program");
writeBlock("G54");
writeln("#100=#900 (read x-offset which was set in G54)");
writeln("#101=0   (safe y for going to workpiece)");
writeln("#102=0   (safe z for going to workpiece)");
writeln('PRINT "x-offset = ";#100;"mm"');
writeln('PRINT "y-offset = ";#101;"mm"');
writeln('PRINT "z-offset = ";#102;"mm"');
writeln('ASKBOOL "Continue with offsets (if no then new offsets can be entered)?" I=2');
writeln('IF #0=0 THEN');
writeln('  ASKFLT "Enter x-offset (0:720)" I=0.0 J=720.0');
writeln('  #100=#0');
writeln('  PRINT "x-offset = ";#100;"mm"');
writeln('  ASKFLT "Enter y-offset (0:420)" I=0.0 J=420.0');
writeln('  #101=#0');
writeln('  PRINT "y-offset = ";#101;"mm"');
writeln('  ASKFLT "Enter z-offset (-50:0)" I=-50.0 J=0.0');
writeln('  #102=#0');
writeln('  PRINT "z-offset = ";#102;"mm"');
writeln('ENDIF');
writeln("");
```

## Explanation

```JavaScript
writeln("");
```

Adds an empty line because the string **""** is empty.

```JavaScript
writeComment("Initial section");
```

Write a comment line into the \*.nc file. The result in the file will look like this (the round brackets are added be the function *writeComment* to the text) and KINETIC-NC interprets this as a comment:

```G-code
(Initial section)
```
Below lines add variables **#100, #101, #102** to the \*.nc file:

```JavaScript
writeComment("So we need to switch to G54 before running the program");
writeBlock("G54");
writeln("#100=#900 (read x-offset which was set in G54)");
writeln("#101=0   (safe y for going to workpiece)");
writeln("#102=0   (safe z for going to workpiece)");
```
 
The values stored in these variables can be used later in the \*.nc file, e.g. for traversing like so:

```G-code
(Go to safe start position)
G53
G0 Z=#102
G0 Y=#101
G0 X=#100
```

In the past I added those lines always manually at the beginning of the \*.nc file in order to move safely to the workpiece without crashing into any clamps. For my typical setups I then just needed to adapt the initial x-coordinate in the initial section.

In order to overcome this, it is now **completely automated based on the settings of G54**. This is the workpiece (stock) offset in most of my setups. KINETIC-NC stores the coordinates of the offsets in non-volatile variables (they are saved across sessions even when the machine is off). The offsets for x, y, and z are stored in the variables #900, #901, #902 respectively.

I normally use only the x-coordinate of the offset (#900) because this fits for 90% of my setups. So in the initial section #900 is assigned to #100 which is used in the subroutine (see below) to do the safe movement to/from the workpiece. I could have used #900 directly in the subroutine, but this way I have more flexibility, for example in cases where I do not want to use the G54 offsets.

When the G-code requests to change the tool, the spindle needs to drive back to machine zero, the new tool (inserted manually) needs to be measured and later the spindle should go back to the workpiece.

This line from the original post-processor:

```JavaScript
writeBlock("T" + toolFormat.format(tool.number), mFormat.format(6));
```

writes the following into the \*.nc file:

```G-code
N1000 T8 M6
```

Which means the command in line number 1000 (for example), tool number 8 (for example) and **M6** is tool change. So I know when the post-processor writes this line a tool-change will happen. Thus I surrounded it (shown below) by two lines which trigger a subroutine that performs the safe traversal to/from machine zero.

So I added a safe path (by calling a subroutine) which moves the spindle automatically according to these variables before and after each tool change:

```JavaScript
writeComment('Go to safe position before tool change');
writeln('M98 P1234 (call subroutine 1234)');
onCommand(COMMAND_COOLANT_OFF);
writeBlock("T" + toolFormat.format(tool.number), mFormat.format(6));
writeComment('Go to safe position after tool change');
writeln('M98 P1234 (call subroutine 1234)');
```

In the G-code this renders to:

```G-code
(Go to safe position before tool change)
M98 P1234 (call subroutine 1234)
N25 M9
N30 T18 M6
(Go to safe position after tool change)
M98 P1234 (call subroutine 1234)
```

As can be seen also the coolant off (M9) needed to be wrapped. Found this by testing.

> [!NOTE]  
>  A tool change requires the tool to be measured again. For this I added the G79 command to the [M66](M66.txt) macro, which resides in the following folder on the PC:

    C:\ProgramData\KinetiC-NC\macros

In KINETIC-NC the M66 macro is called when there is an M6 command (tool change) in the G-code and the machine has no automatic tool change capability. **So in order for this version of the post-processor to work smoothly you need to update your M66.txt accordingly.**

## Subroutines

KINETIC-NC allows following subroutine syntax:

 * M98 &rarr; call subroutine
 * P1234 &rarr; label of the subroutine (caller syntax)
 * O1234: &rarr; label of the subroutine (subroutine function starts here)
 * M99 &rarr; exit subroutine

The calls for adding the subroutine G-code within the post-processor look like this:

```JavaScript
writeComment("Subroutine 1234");
writeln("O1234:");
writeln("G53 (machine coordinates)");
writeln("G0 (go fast)");
writeln("Z=#102 (safety height)");
writeln("Y=#101");
writeln("X=#100");
writeln("G54 (workpiece coordinates)");
writeln("M99 (End Subroutine 1234)");
```

Which would render in the \*.nc file as:

```G-code
(Subroutine 1234)
O1234:
G53 (machine coordinates)
G0 (go fast)
Z=#102 (safety height)
Y=#101
X=#100
G54 (workpiece coordinates)
M99 (End Subroutine 1234)
```

The subroutine uses the variables defined at the beginning of the \*.nc file. So I just need to update these variables once. Then for all subsequent tool changes there should be a safe path "to and from". When I use the same code but have a new workpiece clamped at another position, I again just update the G54 offset(s).

The subroutine call can also point to an external file (not used in this post processor):
```
CALL "SUBD9@plug.txt"
```

In the same way, more functionality can be added to the post-processor or different *dialects* of the same post-processor could be created depending on the requirements.

## Example for using the jump labels

KINETIC-NC allows skipping portions of the code by using the **SKIP** command. This comes in handy when in a longer NC program a certain section should be done later again, but some other operations not. In order to support this upfront, the respective code is added by the post-processor as comments, so that it just has to be uncommented when being used.

> [!NOTE]  
> A label cannot exist on its own unless it has been defined in by the SKIP command before.

> [!NOTE]  
>  Labels need to use other characters than those used in G-code

Following lines show an example of how this is prepared in the G-code by the post-processor:

```G-code
(Uncomment to skip until specified label)
(Skip label must exist, than jump label is accepted)
(SKIP Q0001)
...
...
....
(Q0001:)
```

If needed the code can be activated by editing the code like:

```G-code
SKIP Q0001
...
...
....
Q0001:
```

The code between the SKIP command and the label is not executed. The SKIP command is typically used with **IF..THEN** constructs.

> [!NOTE]  
> The colon is needed only at the label itself. In the line where the SKIP command is, it is not allowed.

## Example using REPEAT/NEXT

KINETIC-NC allows loops over portions of the G-code. In order to facilitate this feature, the **REPEAT** and **NEXT** keywords are automatically inserted close to the top and at the end of each section.

The following code snippet from the thread example file shows this:
    
```G-code
(Chamfer)
M98 P1234 (call subroutine 1234)
N1605 M9
N1610 T13 M6
M98 P1234 (call subroutine 1234)
N1615 S5000 M3
N1620 G54
(Edit repeat count according to needs)
REPEAT=1
N1625 M9
N1635 G0 X27.95 Y12.5
N1640 G0 G43 Z15 H13
N1645 G0 Z5
N1650 G1 Z-2.5 F333.3
N1655 G1 X28.75 F1000
N1660 G3 X21.25 I-3.75 J0
N1665 G3 X28.75 I3.75 J0
N1670 G3 X27.6126 Y15.1901 I-3.75 J0
N1675 G1 X27.0553 Y14.6162
N1680 G0 Z5
N1685 G0 X40.6 Y12.5
N1690 G1 Z-2.5 F333.3
N1695 G1 X41.4 F1000
N1700 G3 X38.6 I-1.4 J0
N1705 G3 X41.4 I1.4 J0
N1710 G3 X39.2421 Y13.6771 I-1.4 J0
N1715 G1 X39.6752 Y13.0045
N1720 G0 Z15
NEXT
```

Just change the number after the **REPEAT** keyword from **1** to a bigger number and the section should be executed the respective number of times.

> [!TIP]  
> In order to debug the post-processor code in FUSION 360, tick the "Open Nc-file in editor" option in the post-processor before saving. When the code change fails, the corresponding error message(s) are shown in the editor. Another indication that something went wrong, is an empty properties window at the lower right of the post-processor dialog.
