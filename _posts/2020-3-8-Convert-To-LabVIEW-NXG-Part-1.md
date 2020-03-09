---
layout: post
title: Let's Convert A LabVIEW Project to LabVIEW NXG! (Part 1)
---

![Let's convert!]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/title.png)

This is the first of a two-part blog documenting the experience of converting a small/medium sized LabVIEW project to LabVIEW NXG.

The source code for the original [LabVIEW project](https://github.com/dataflowg/dataflow-dj/releases/tag/v0.1.0) and converted [LabVIEW NXG project](https://github.com/dataflowg/dataflow-dj-nxg) are linked on github.

# Background

The plan was to convert the first release of [Dataflow DJ](https://github.com/dataflowg/dataflow-dj) to LabVIEW NXG 4.0, the goal being playback of two tracks with some simple mixing. This project was chosen as it's relatively small, though complex enough to test a range of features (subpanels, classes, sound output, DVRs, DLLs, signal processing, etc). It's also quite demanding in terms of CPU, so would be a good test of NXG's run-time performance.

The knew the UI probably wouldn't port nicely, so the focus was on achieving basic functionality. I did briefly consider trying to port the latest Dataflow DJ release, but it's split across multiple projects, with multiple static and dynamic PPLs. That was painful enough to implement in LabVIEW - I wasn't even going to attempt it in NXG!

[![Version 0.1 of Dataflow DJ.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/LV-DataflowDJ.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/LV-DataflowDJ.png)

Here's a quick look at the project under LabVIEW 2019. A few classes, a few libraries, some loose VIs and controls, and a top level VI. It's all pure LabVIEW, save for some DLL calls to load MP3s. The project is self contained, with no dependencies beyond vi.lib.

[![Dataflow DJ LabVIEW project.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/LV-DataflowDJ-Project.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/LV-DataflowDJ-Project.png)

# Code Conversion

First step is the Code Conversion Utility. There's some conversion options which can be set, but I stuck with the defaults for this experiment. The vi.lib option would've helped with some missing sound VIs that we'll run into later, but I wanted to see how far things went with the vanilla configuration.

[![Code Conversion Utility settings page.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/CCU-Settings.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/CCU-Settings.png)

Interesting tooltip - is 5335 really the recommended connector pane size in NXG? New NXG VIs all have the 4224 pattern by default. Presumably this is the recommendation for instrument drivers, which would be a common conversion utility use case.

[![Setting to enlarge VI connector panes.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/CCU-Settings-5335.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/CCU-Settings-5335.png)

The LabVIEW project file was added for conversion, and the preview indicated no errors or missing files. After a couple of minutes the conversion completed (without crashing or hanging I hasten to add, which was a problem in previous NXG versions). The converted project is then immediately opened, so it looks like it worked. Time to check the conversion report.

[![Setting to enlarge VI connector panes.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/CCU-Preview.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/CCU-Preview.png)

## Code Conversion Errors

It didn't take long to find the first show stopper - the main DJ Interface VI failed to convert. At this point it would've been nice if the conversion utility highlighted files which failed to convert, rather than needing to dig into the conversion report. There are filtering tools available for looking through the conversion report, so finding failures is easy enough (but one still has to go looking). In this case the report highlights an EventDataNode, and a potentially corrupt VI. The VI certainly isn't corrupt, so it's probably the event structure or event registration node in that VI. Time to fire up LabVIEW.

[![Conversion report output.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/NXG-Conversion-Report.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/NXG-Conversion-Report.png)

The process for narrowing down the fault was the usual 'remove code until works, then add it back until it breaks' approach. The first test removed the event structure entirely, then the project was reconverted. There were no conversion errors this time, so it seems the event structure or perhaps a particular event is at fault. The next test removed all of the registered events, but kept the structure and code in place, and this also converted without errors.

After a slow process of elimination, the root cause of the conversion error is dynamically registered filter events (Key Down?, Panel Close?, etc). This VI snippet highlights the problem. If those dynamically registered filter events are removed from the event structure, the VI converts successfully.

[![LabVIEW 2019 snippet which will fail to convert to LabVIEW NXG.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/LV-Dynamic-Filter-Event-Reg-Snippet.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/LV-Dynamic-Filter-Event-Reg-Snippet.png)

## Code Conversion Successes

One issue the conversion utility dealt with was circular dependencies between libraries. I knew Dataflow DJ had some circular dependencies (VIs using typedefs from other libraries, and vice versa). The utility shifted the offending VIs and controls to the new Merged and MergedDependencies libraries to fix the issue. Kudos.

By the way you can view circular dependencies in LabVIEW using the VI Hierarchy view. The links appear as curved wires connecting the offending VIs/libraries.

[![Oh no! Circular dependencies!.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/LV-VI-Hierarchy.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/LV-VI-Hierarchy.png)

## This Feature Is Not Supported In This Version Of LabVIEW NXG

With that erroneous VI fixed, all project files are now present and accounted for. The next step I took was fixing issues highlighted in the conversion report, which are almost all some variant of *this feature is not supported in this version of LabVIEW NXG*. I don't want to provide detail on how every entry was fixed, but will address some of the standout issues and workarounds required.

Before diving in, I was curious to see how well the main interface converted. It's not terrible! (though suffers a [knob shrinkage](https://twitter.com/Dataflow_G/status/1060542351528013824) problem)

[![The initial DJ Interface UI conversion.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/NXG-DJ-Interface.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/NXG-DJ-Interface.png)

# Make It Work

The process of fixing the conversion errors was slow to start, as I was still learning what LabVIEW NXG 4.0 was and wasn't capable of. Drag and drop? Nope. Dynamic events from control references? Nope. Execution systems? Nope.

Then came the more subtle differences I wasn't expecting. Unbundle caption text from a control reference? That needs two property nodes now. Bundle a cluster element on a class wire? That needs three property nodes. Why is the knob control bounding box so much bigger than the graphic? Hang on, no *Value* property!? What was going on?

## All References Are Equal, But Some References Are More Equal Than Others

One of these reference types is a NumericTextBox. The other is a NumericTextBox. See the difference? Maybe the context help gives a clue? What about the item tab on the configuration pane? Everything indicates they're both a NumericTextBox. They must be the same then! So why does only one of them have a subset of properties?

[![Two NumericTextBox references, but only one has all the properties.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/NXG-Reference-Compare.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/NXG-Reference-Compare.png)

Here's a hint - they have different underlying numeric types. Or rather, one of them has no numeric type.

The reference on the left was created as a constant from a NumericTextBox front panel control reference, its type set to double. The reference on the right was created as a reference type specifier constant from the palette, and its type selected using the type picker. Without knowing the provenance of each of the references, there's no indication they are different. One can check the actual type by opening the saved VI in a text editor (FYI they're stored as XML now), and check that the NumericTextBox is a NumericTextBox**Double**Proxy, and not a typeless NumericTextBoxProxy, but c'mon.

This is something I can *kinda* understand on a behind the scenes level, but for an end user it makes for a very confusing and ultimately frustrating experience. This user was no exception.

### Where's The Value?

Following on from the reference type perculiarities above, I ran into further problems accessing the value property. This time from control references that actually *have* a numeric type. Here's a simplified example of the problem. There are references to two control types, a NumericTextBox (Numeric) and a Slider (Slider). Both have a double type, and so both have a value property. The build array function implicitly type casts both reference types to their nearest ancestor, the Numeric class. The problem here is this cast strips the numeric type, so it's now a typeless Numeric and not a double Numeric. So we lose access to the Value property (without converting back to the correct control + numeric type reference).

[![Different control references with a double type, cast to a more general Numeric lose the Value property.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/NXG-Reference-Value-Property-Missing.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/NXG-Reference-Value-Property-Missing.png)

In Dataflow DJ, references to every UI control are stored in a variant attribute style map, and accessed when a value change message is received from the audio engine (e.g. *Deck1.PlaybackPosition*). The original LabVIEW code handling UI updates receives the value change message, reads the corresponding control reference out of the variant attribute map, then writes the new value to the the Value property of that control. This method can handle a whole range of controls and data types (sliders, knobs, waveforms, strings).

[![The Value property in LabVIEW works regardless of the control reference type.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/LV-Reference-Value-Write.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/LV-Reference-Value-Write.png)

With NXG, the ability to usefully write to the Value property in this way is gone, unless the control reference is converted to the exact UI control type *and* numeric representation. It might be possible to manage this with a map per control / value type or some other architectural redesign, but that was beyond the scope of this experiment.

I did manage to find a workaround in the form of `Set Control Value`, which writes a variant input to a named control, no reference type necessary. I don't know if this is the expected way to do things in NXG, or just happens to be the one which works. It's also unclear if it's more or less performant, but it worked so I stuck with it.

## Diagram Flimflam

OK, so not a lot of properties are supported. For those which do exist, multiple additional property nodes can be required to access them. Take the contrived example below. If one wants to set all of the scale minimums of an intensity plot to 0 (leaving the maximum in place), this is the (only?) way it can be done in NXG. Every reference requires a property node, and in some cases clusters require their own property nodes too. The error wire at the end ensures the update order. The equivalent LabVIEW code is a single property node with three inputs.

[![NXG's property node is limited.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/Intensity-Graph-Property-Compare.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/Intensity-Graph-Property-Compare.png)

This property node requirement extends to class members. If a class contains a cluster, elements of that cluster can not be directly accessed from a property node on the class wire. Instead the cluster needs to be unbundled, and a second property node used to access the elements. Writing to cluster elements requires even more property nodes and wires.

[![NXG's class property node is limited too.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/Class-Property-Compare.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/Class-Property-Compare.png)

While this property node nitpick doesn't affect functionality, all of these extra property nodes add to the diagram space. I really hope direct access to nested properties is coming in a future version of LabVIEW NXG, otherwise I'll have to seriously consider getting one of these monitors.

[![21:9 ultrawide monitor.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/ultrawide.jpg)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/ultrawide.jpg)

(One could/should use IPEs, but they are additional diagram bloat if performance or memory safety isn't a concern. NXG's IPEs even more so, they're enormous.)

## The Event Structure

The event structure provided a few interesting quirks. Dynamic registration for control references is out, so the once simple dynamic event registration of an array of UI control references became a tedious effort in configuring 50+ event sources across several event cases. Also one can't register a Mouse Down? filter event for different control types, or even *same* control types with different data types. So there's a bit of [WET](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself#DRY_vs_WET_solutions) going on when configuring events.

[![The NXG event configuration dialog.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/NXG-Event-Config.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/NXG-Event-Config.png)

And that event config dialog is too damned small. I can only view a few event sources and configured events at once (and the dialog in the picture has been resized to its maximum, the default is even *smaller*). This dialog is somehow worse than the original event configuration dialog in LabVIEW (which was thankfully overhauled). Here's hoping the NXG event config dialog sees a similar overhaul.

[![The old but less worse LabVIEW event configuration dialog.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/LV-Event-Config-Old.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/LV-Event-Config-Old.png)

What's also puzzling is the ability to add events to the event structure from the Item tab on the right. Once added, how are they configured? Oh right, on the diagram with that tiny dialog. Why even add them on the Item pane?

## External library bitness fitness

It's mentioned multiple times in the manuals, but any calls to 32-bit DLLs will not work under LabVIEW NXG as it's 64-bit only. Not a problem in this project, as the MP3 loading DLL can to recompiled to 64-bit. It may be a problem if a hardware driver or library is only available as a 32-bit DLL. In the interests of getting Dataflow DJ running, MP3 loading was disabled and only WAV loading (via the built-in sound library) was used.

## O LLB VIs, Where Art Thou?

There are a few VIs from lvsound2.llb which are not visible in the LabVIEW Sound palette, but used by Dataflow DJ to query device information. Unfortunately these VIs are no longer included with NXG (though the underlying functions are there in lvsound2.dll). This is where the Code Conversion Utility's Convert files referenced from vi.lib option is handy, porting over VIs which don't exist in NXG.

I gave it a quick go and it works as advertized, though there's one major pitfall - if you don't know the password for protected VIs in vi.lib, they won't convert. So if you're relying on some hidden vi.lib gems, make sure you know the password (or that they aren't password protected).

In the end I didn't use this conversion option, and just assumed a single audio device would be used (so no headphone cueing). Looking back I don't know if trying to output to two devices simulatenously would've been too much of a performance issue anyway...

## Subpanels vs. Panel Containers

I was really hoping the Code Conversion Utility would perform a 1:1 replacement of subpanels with panel containers, but that wasn't the case. No matter, it should be simple enough to replace them manually...

### Is Your VI Running?

For a VI to be inserted into a panel container, it needs to be in a running state. There are two common ways to start a VI asynchronously - `Run VI` and `Start Asynchronous Call`. Both of these methods return a VI reference, which can then be wired up to the insert method for a subpanel in LabVIEW, or a panel container in NXG. Both of these methods work for LabVIEW subpanels, but only one of these methods works with a panel container in LabVIEW NXG. Guess which one I used in Dataflow DJ?

[![Both methods work in LabVIEW, but only Run VI works in NXG.]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/Subpanels-PanelContainers.png)]({{ site.baseurl }}/images/Convert-To-LabVIEW-NXG-Part-1/Subpanels-PanelContainers.png)

This had me stumped for a while, and I'm still unclear as to why one method works and the other doesn't (maybe different thread pools?). The original LabVIEW code opens a VI reference with the async flag (0x80) and uses the asynchronous call VIs to start the VI running. The converted NXG code can successfully run the VIs using this same method, but the panel container was insistent the VI wasn't running and refused to load the VI into the panel container (error 1690).

If I opened the NXG VI after it had been started asynchronously, it was definitely running. Querying the Execution State property of the VI reference showed it was running. So in a final act of desperation I replaced the asynchronous call with the Run VI node and... it worked. The documentation for panel containers is a little sparse - it only states a VI has to be running for it to be placed in the panel container, and nothing on how the VI should be started. Further, there are no examples which show the 'right' way to use panel containers.

# Part 1 Conclusion

The LabVIEW 20xx to LabVIEW NXG code conversion was actually fairly painless (save for the event structure issue). The most difficult part of the conversion was in working around (or at least understanding) the shortcomings of NXG itself. It's still young(-ish) and under constant development, so I expect those pain points to go away in future versions.

Part 2 will cover performance optimizations, unicode workarounds, general debugging, and the Shared Library Interface (ugh). Stay tuned!
