# Fusion 360 KINETIC-NC Post-Processor
An Autodesk Fusion 360 post-processor for the [HIGH-Z S-720/T](https://www.cnc-step.de/cnc-fraese-high-z-s-720t-kugelgewindetrieb-720x420mm) milling machine running the [KINETIC-NC](https://www.cnc-step.de/cnc-software/kinetic-nc-netzwerk-steuerungssoftware/) software, which both are from [CNC-Step](https://www.cnc-step.de).

The post processor is based on the classical format [RS-274D](https://en.wikipedia.org/wiki/G-code) also called [G-code](https://en.wikipedia.org/wiki/G-code). The original version of this RS-274D implementation for the FUSION 360 post-processor comes from [Benezan Electronics](http://www.benezan-electronics.de/index.html). You can download their post-processor [here](http://www.benezan-electronics.de/downloads/Autodesk_HSM_beamicon2.zip). For convenience I saved a copy in this repository.

The reason why the Benzean post-processor works for the KINETIC-NC software is that the KINETIC-NC software is based on the BEAMICON2 software of Benezan.

Thus I took above postprocessor and ran it successfully on FUSION 360 for my HIGH-Z S-720/T machine which is conrolled via the KINETIC-NC software. 

I added the some entries to the post-processor in order to make it more convenient for my typical operations. The post-processor is written in JavaScript and the documentation of the existing classes, functions, etc. is described in the [Autodesk CAM Post Processor Documentation](https://cam.autodesk.com/posts/reference/index.html).

Adding some commands to the post-processor is quite easy, as I needed to use only two already existing functions. The main function needed is **writeln()** which comes with the Autodesk JavaScript API and the second one is **writeComment()** which is a wrapper around **writeln()** that just adds brackets before and after the text (this is the comment format understood by KINETIC-NC).

