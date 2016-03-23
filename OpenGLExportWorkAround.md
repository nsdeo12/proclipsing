# Introduction #

This is a very helpful write up by Eric Socolofsky about how to deal with issues you may encounter when exporting OpenGL Projects with Proclipsing.

# Details #

Proclipsing trouble, especially with OpenGL? Read this.
in Integration and Hardware  •  4 days ago
I'm a long time user of Proclipsing, and before that EclipseP5Exporter.  Proclipsing is a super useful tool, and opens the gate for Processing users to an even more useful tool, namely Eclipse.

However, as time has gone on, I've run into more and more trouble exporting applications from Eclipse via Proclipsing.  I finally hit a wall and had to haul my ass over it, so wanted to share with you what I learned in the process.

Note: I'm on OSX 10.6, so your mileage may vary on other platforms.


Where Proclipsing works just fine
When you're not using other libraries (including OpenGL, which is used by default from 2.0 alpha 6 forward), Proclipsing seems to work very well.  It's when you have libraries involved that things start to get messy.


Where Proclipsing breaks
I've noticed that Proclipsing copies files into Eclipse.app when it first exports an application (after being installed).  Within Eclipse.app > Contents > MacOS, Proclipsing creates a '/lib' folder, and dumps a bunch of files from...somewhere.  I'm not sure where these files come from exactly, but they include OpenGL jars and natives.  These files stashed in Eclipse.app are then copied to applications exported from Eclipse by Proclipsing.  (Note: "natives" are files that connect java to the operating system for certain functionality like serial, video, and graphics card access, which is needed by OpenGL.  These files are named **.jnilib for OSX,**.dll for Windows, and **.so for Linux.)**

When you create new Proclipsing Projects in Eclipse and target new versions of Processing, Proclipsing will copy the jars from the libraries within the specified version of Processing into the /lib folder of your project. I've found that this generally works as intended.  When it doesn't, you can often fix things up by editing your Project>Properties>Java Build Path, ensuring that required jars are on the build path and that the "Native library location" for each jar that uses natives is set to the path of those natives.

However, when exporting an application with libraries, specifically with OpenGL, it seems that Proclipsing copies the files it stored inside Eclipse.app into your application, regardless of what's inside your project's /lib folder.


How to fix it
I found my answers by poking around inside a simple OpenGL application exported from Processing 2.0a6 (which works), and comparing (via a free software called FileMerge) with the contents of a Proclipsing-exported application using 2.0a6 (which doesn't work).  There are a couple of major differences to correct in the Proclipsing export to get it to run like the Processing export.  To fix up your own Proclipsing exports, I suggest you export an application from Processing and cannibalize it.

1. Package the correct jars
Open up your Processing-exported OpenGL application via right-click>Show Package Contents (on OSX), and navigate to Contents/Resources/Java.  You will see a jar with the name of your application (we'll call it MyProcessingExport.jar), and six other jars (as of v2.0a6).  These jars are core.jar, gluegen-rt.jar, jogl.all.jar, opengl.jar, and two other platform-specific jars that contain GL natives (e.g. for OSX: gluegen-rt-natives-macosx-universal.jar and jogl-all-natives-macosx-universal.jar).

Open up your Proclipsing-exported OpenGL application the same way and navigate to the same /Java folder.  You'll likely see a whole bunch of other files.  Delete all of them except the jar with the name of your exported application (we'll call it MyProclipsingExport.jar).

Copy all of the jars from the Processing export **except** MyProcessingExport.jar into the /Java folder within your Proclipsing export.

2. Edit your info.plist file
The info.plist file in an OSX application tells OSX how to run an application.  For Java applications, it includes information about the JVM to use, whether to use 32- or 64-bit mode, what command-line JVM options to use, and the class path to other jars.  Info.plist is an xml file, and can be edited in any text editor.

Proclipsing forces the JVM to run in 32-bit mode to get around issues with "video and most other native libraries" (per a comment in the Proclipsing-generated info.plist file). I think this may be an outdated issue, and I've found that 2.0a6 OpenGL applications run twice as fast in 64-bit mode, so let's eliminate that restriction.  Within 

&lt;key&gt;

LSArchitecturePriority

&lt;/key&gt;

, add 

&lt;string&gt;

x86\_64

&lt;/string&gt;

 to the list of JVM architectures.  You can also eliminate PowerPC if you're not running on a very old machine.  You should now have:
Copy code
```
<key>LSArchitecturePriority</key>
<array>
  <string>x86_64</string>
  <string>i386</string>
</array>
```
You should also specify JVM 1.6, as that's now the preferred JVM for Processing applications.  Proclipsing defaults to 1.5.  Change the value below 

&lt;key&gt;

JVMVersion

&lt;/key&gt;

 to 1.6**:
Copy code**

&lt;key&gt;

JVMVersion

&lt;/key&gt;




&lt;string&gt;

1.6**&lt;/string&gt;


We specified the JVM architecture (32- or 64-bit mode) above, so delete the JVMArchs key and the array below it:
Copy code
```
<key>JVMArchs</key>
<array>
  <string>i386</string>
  <string>ppc</string>
</array>
```
Finally, ensure your**

&lt;key&gt;

ClassPath

&lt;/key&gt;

 only has the jars we placed inside the application.  Simplest way to do this is to copy this 

&lt;string&gt;

 value from the info.plist in the Processing-exported application over the 

&lt;string&gt;

 value in the Proclipsing-exported application.  We then have to ensure the line references the name of the Proclipsing-exported application jar (e.g. MyProclipsingExport.jar), not the jar export from Processing (e.g. MyProcessingExport.jar).  You should have something like:
Copy code
```
<key>ClassPath</key>
<string>$JAVAROOT/MyProclipsingExport.jar:$JAVAROOT/gluegen-rt-natives-macosx-universal.jar:$JAVAROOT/gluegen-rt.jar:$JAVAROOT/jogl-all-natives-macosx-universal.jar:$JAVAROOT/jogl.all.jar:$JAVAROOT/opengl.jar</string>
```
(Note that if you have other jars in your project, they should be referenced here as well.)


Run your fixed application
Double-click it.  It should now run, and possibly twice as fast as before since it's now running in 64-bit mode.  Yay!

If not (Boo...), open up Console.app (in Applications>Utilities) and see if you get any clues from there about missing classes or files.  I've found that in some circumstances, Proclipsing does not package your /data folder into the exported application.  Not sure why yet, but when this happens Console.app can help you see the failed file loads. The path at which a Java application starts looking for external resources (e.g. fonts, images, data or config files) is the path from which it is launched.  For example, if your application is looking for a file at path "data/somefile.txt", in the case of a double-clickable application, your /data folder should be in the same folder as MyExportedApp.app.

Phew.  That is a synopsis of a couple days of slogging through this stuff, and I've learned more than I reported here.  So if you have questions or clarifications please post them and I'll try to help.  Also, I'm sorry I don't have instructions for OSes other than OSX; I'm not sure how Java applications are bundled on other platforms.  If you have tips, please add them as comments.


Finally, I understand that the Proclipsing team is not really supporting Proclipsing anymore because of the Processing Eclipse Plugin, a Processing-endorsed alternative to Proclipsing.  Unfortunately, the Eclipse Plugin seems to be dead on the vine; the last edit was January 2011, and the bottom of that wiki page reads

While the UI for an export wizard is in place, it doesn't fully function.  A partial export will take place, and log a bunch of messages to the console but nothing functional will have happened. For now, you'll have to use the PDE as an external editor.

I totally understand the Proclipsing team's decision to stop supporting a tool that must be updated every time Processing is updated -- it's hard work.  If anyone out there wants to take up the mantle and fix these issues, I'm sure they'd welcome the assistance.

Hope this helps!
-Eric