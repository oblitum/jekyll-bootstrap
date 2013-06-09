---
layout: page
title: "Interception"
description: ""
---
{% include JB/setup %}

###What’s Interception?

A programming interface for intercepting input device communication.

###What can I do with that?

Currently, with Interception you are able to intercept and transform input data from keyboard and mouse.
Support is still Windows only (from Windows 2000 to Windows 7).

###I can use Windows hooks for that, so, what’s up?

Yes, you can, but...

  - You're not able to identify from which device some event is coming when you connect multiple keyboard/mouse.
  With generic user-mode hooks you are able to identify that a keyboard/mouse event has arrived, but not from which keyboard/mouse it’s coming.  
  - With Windows Raw Input you can identify but cannot _intercept_ (and then mutate) input.  
  - You can't intercept CTRL-ALT-DELETE with user-mode hooks, for example, and do other stuff like peering while at the login screen.
  - You can't hook old DirectInput games.

###Ok, but, how do you achieve this?

The Interception API provides a simple interface of communication with kernel-mode components, and those components are powerful.
They intercept data in a early stage, before it reaches the core of OS input processing.

###I have a x64 Windows box, I’ve heard that it cannot load kernel-mode components indiscriminately.

Yes, indeed. The Interception core is built upon properly signed drivers, so that’s not a problem!

###Ok then, let’s get to work.

First, you must download the Interception core installation tool: [install-interception](https://github.com/downloads/oblitum/Interception/install-interception.exe).
This tool must be run from an administrator command line (you must run cmd as administrator). Just run it without arguments to receive instructions.

Second, you must link against the Interception library, which is provided [here](https://github.com/downloads/oblitum/Interception/Interception.zip)
as a convenient package, or you may build it from [sources](https://github.com/oblitum/Interception). Static and dynamic libraries are provided and
the x86 version do not depend on the VC10 runtime.

Third, you must include interception.h in your applications.

The following sample shows how to intercept the x key and turn it into y:

<pre>
<span class="Include">#include </span><span class="String">&lt;interception.h&gt;</span>
<span class="Include">#include </span><span class="String">&quot;utils.h&quot;</span>

<span class="Structure">enum</span> ScanCode
<span class="lv16c">{</span>
    SCANCODE_X   <span class="op_lv16">=</span> <span class="Number">0x2D</span><span class="op_lv16">,</span>
    SCANCODE_Y   <span class="op_lv16">=</span> <span class="Number">0x15</span><span class="op_lv16">,</span>
    SCANCODE_ESC <span class="op_lv16">=</span> <span class="Number">0x01</span>
<span class="lv16c">}</span><span class="op_lv0">;</span>

<span class="Type">int</span> main<span class="lv16c">()</span>
<span class="lv16c">{</span>
    InterceptionContext context<span class="op_lv16">;</span>
    InterceptionDevice device<span class="op_lv16">;</span>
    InterceptionKeyStroke stroke<span class="op_lv16">;</span>

    raise_process_priority<span class="lv15c">()</span><span class="op_lv16">;</span>

    context <span class="op_lv16">=</span> interception_create_context<span class="lv15c">()</span><span class="op_lv16">;</span>

    interception_set_filter<span class="lv15c">(</span>context<span class="op_lv15">,</span> interception_is_keyboard<span class="op_lv15">,</span> INTERCEPTION_FILTER_KEY_DOWN <span class="op_lv15">|</span> INTERCEPTION_FILTER_KEY_UP<span class="lv15c">)</span><span class="op_lv16">;</span>

    <span class="Repeat">while</span><span class="lv15c">(</span>interception_receive<span class="lv14c">(</span>context<span class="op_lv14">,</span> device <span class="op_lv14">=</span> interception_wait<span class="lv13c">(</span>context<span class="lv13c">)</span><span class="op_lv14">,</span> <span class="lv13c">(</span>InterceptionStroke <span class="op_lv13">*</span><span class="lv13c">)</span><span class="op_lv14">&amp;</span>stroke<span class="op_lv14">,</span> <span class="Number">1</span><span class="lv14c">)</span> <span class="op_lv15">&gt;</span> <span class="Number">0</span><span class="lv15c">)</span>
    <span class="lv15c">{</span>
        <span class="Conditional">if</span><span class="lv14c">(</span>stroke<span class="op_lv14">.</span>code <span class="op_lv14">==</span> SCANCODE_X<span class="lv14c">)</span> stroke<span class="op_lv15">.</span>code <span class="op_lv15">=</span> SCANCODE_Y<span class="op_lv15">;</span>

        interception_send<span class="lv14c">(</span>context<span class="op_lv14">,</span> device<span class="op_lv14">,</span> <span class="lv13c">(</span>InterceptionStroke <span class="op_lv13">*</span><span class="lv13c">)</span><span class="op_lv14">&amp;</span>stroke<span class="op_lv14">,</span> <span class="Number">1</span><span class="lv14c">)</span><span class="op_lv15">;</span>

        <span class="Conditional">if</span><span class="lv14c">(</span>stroke<span class="op_lv14">.</span>code <span class="op_lv14">==</span> SCANCODE_ESC<span class="lv14c">)</span> <span class="Statement">break</span><span class="op_lv15">;</span>
    <span class="lv15c">}</span>

    interception_destroy_context<span class="lv15c">(</span>context<span class="lv15c">)</span><span class="op_lv16">;</span>

    <span class="Statement">return</span> <span class="Number">0</span><span class="op_lv16">;</span>
<span class="lv16c">}</span>
</pre>

There are some concepts you need to know to use Interception well:

  - Contexts  
    You need to create a context of communication with the core components, most functions will require it.
  - Filters  
    Filters are a way to indicate what kind of events you want to intercept, for example, in the last sample
    we were interested in simple key up’s and key down’s, some special keys would not trigger events through this
    filter, like the DELETE key, because it needs the E0 flag too. If you do not set a filter for at last one
    device, you won’t be notified of any events.
  - Device selection predicate  
    The interception\_set\_filter function has three parameters, the context of communication, a function pointer
    and a desired filter. This second parameter, the function pointer, is a device selection predicate, a function
    that receives a device id (like INTERCEPTION_KEYBOARD(0), INTERCEPTION_KEYBOARD(1), etc) as parameter and returns
    true if the device id passed is one of a device that must be filtered through the chosen filter, or false for the
    devices that are not to be filtered through this filter. 
    So the interception_set_filter works by scanning all possible devices, and using the provided predicate as the
    criteria to know for which devices the provided filter should be applied.

The following sample shows how to invert the vertical mouse axis:

<pre>
<span class="Include">#include </span><span class="String">&lt;interception.h&gt;</span>
<span class="Include">#include </span><span class="String">&quot;utils.h&quot;</span>

<span class="Structure">enum</span> ScanCode
<span class="lv16c">{</span>
    SCANCODE_ESC <span class="op_lv16">=</span> <span class="Number">0x01</span>
<span class="lv16c">}</span><span class="op_lv0">;</span>

<span class="Type">int</span> main<span class="lv16c">()</span>
<span class="lv16c">{</span>
    InterceptionContext context<span class="op_lv16">;</span>
    InterceptionDevice device<span class="op_lv16">;</span>
    InterceptionStroke stroke<span class="op_lv16">;</span>

    raise_process_priority<span class="lv15c">()</span><span class="op_lv16">;</span>

    context <span class="op_lv16">=</span> interception_create_context<span class="lv15c">()</span><span class="op_lv16">;</span>

    interception_set_filter<span class="lv15c">(</span>context<span class="op_lv15">,</span> interception_is_keyboard<span class="op_lv15">,</span> INTERCEPTION_FILTER_KEY_DOWN <span class="op_lv15">|</span> INTERCEPTION_FILTER_KEY_UP<span class="lv15c">)</span><span class="op_lv16">;</span>
    interception_set_filter<span class="lv15c">(</span>context<span class="op_lv15">,</span> interception_is_mouse<span class="op_lv15">,</span> INTERCEPTION_FILTER_MOUSE_MOVE<span class="lv15c">)</span><span class="op_lv16">;</span>

    <span class="Repeat">while</span><span class="lv15c">(</span>interception_receive<span class="lv14c">(</span>context<span class="op_lv14">,</span> device <span class="op_lv14">=</span> interception_wait<span class="lv13c">(</span>context<span class="lv13c">)</span><span class="op_lv14">,</span> <span class="op_lv14">&amp;</span>stroke<span class="op_lv14">,</span> <span class="Number">1</span><span class="lv14c">)</span> <span class="op_lv15">&gt;</span> <span class="Number">0</span><span class="lv15c">)</span>
    <span class="lv15c">{</span>
        <span class="Conditional">if</span><span class="lv14c">(</span>interception_is_mouse<span class="lv13c">(</span>device<span class="lv13c">)</span><span class="lv14c">)</span>
        <span class="lv14c">{</span>
            InterceptionMouseStroke <span class="op_lv14">&amp;</span>mstroke <span class="op_lv14">=</span> <span class="op_lv14">*</span><span class="lv13c">(</span>InterceptionMouseStroke <span class="op_lv13">*</span><span class="lv13c">)</span> <span class="op_lv14">&amp;</span>stroke<span class="op_lv14">;</span>

            <span class="Conditional">if</span><span class="lv13c">(</span><span class="op_lv13">!</span><span class="lv12c">(</span>mstroke<span class="op_lv12">.</span>flags <span class="op_lv12">&amp;</span> INTERCEPTION_MOUSE_MOVE_ABSOLUTE<span class="lv12c">)</span><span class="lv13c">)</span> mstroke<span class="op_lv14">.</span>y <span class="op_lv14">*=</span> <span class="op_lv14">-</span><span class="Number">1</span><span class="op_lv14">;</span>

            interception_send<span class="lv13c">(</span>context<span class="op_lv13">,</span> device<span class="op_lv13">,</span> <span class="op_lv13">&amp;</span>stroke<span class="op_lv13">,</span> <span class="Number">1</span><span class="lv13c">)</span><span class="op_lv14">;</span>
        <span class="lv14c">}</span>

        <span class="Conditional">if</span><span class="lv14c">(</span>interception_is_keyboard<span class="lv13c">(</span>device<span class="lv13c">)</span><span class="lv14c">)</span>
        <span class="lv14c">{</span>
            InterceptionKeyStroke <span class="op_lv14">&amp;</span>kstroke <span class="op_lv14">=</span> <span class="op_lv14">*</span><span class="lv13c">(</span>InterceptionKeyStroke <span class="op_lv13">*</span><span class="lv13c">)</span> <span class="op_lv14">&amp;</span>stroke<span class="op_lv14">;</span>

            interception_send<span class="lv13c">(</span>context<span class="op_lv13">,</span> device<span class="op_lv13">,</span> <span class="op_lv13">&amp;</span>stroke<span class="op_lv13">,</span> <span class="Number">1</span><span class="lv13c">)</span><span class="op_lv14">;</span>

            <span class="Conditional">if</span><span class="lv13c">(</span>kstroke<span class="op_lv13">.</span>code <span class="op_lv13">==</span> SCANCODE_ESC<span class="lv13c">)</span> <span class="Statement">break</span><span class="op_lv14">;</span>
        <span class="lv14c">}</span>
    <span class="lv15c">}</span>

    interception_destroy_context<span class="lv15c">(</span>context<span class="lv15c">)</span><span class="op_lv16">;</span>

    <span class="Statement">return</span> <span class="Number">0</span><span class="op_lv16">;</span>
<span class="lv16c">}</span>
</pre>

The following sample shows how to intercept and block the CTRL-ALT-DEL sequence:

<pre>
<span class="Include">#include </span><span class="String">&lt;interception.h&gt;</span>
<span class="Include">#include </span><span class="String">&quot;utils.h&quot;</span>

<span class="Include">#include </span><span class="String">&lt;iostream&gt;</span>
<span class="Include">#include </span><span class="String">&lt;deque&gt;</span>

<span class="Structure">enum</span> ScanCode
<span class="lv16c">{</span>
    SCANCODE_ESC <span class="op_lv16">=</span> <span class="Number">0x01</span>
<span class="lv16c">}</span><span class="op_lv0">;</span>

InterceptionKeyStroke nothing <span class="op_lv0">=</span> <span class="lv16c">{}</span><span class="op_lv0">;</span>
InterceptionKeyStroke ctrl_down <span class="op_lv0">=</span> <span class="lv16c">{</span><span class="Number">0x1D</span><span class="op_lv16">,</span> INTERCEPTION_KEY_DOWN<span class="lv16c">}</span><span class="op_lv0">;</span>
InterceptionKeyStroke alt_down  <span class="op_lv0">=</span> <span class="lv16c">{</span><span class="Number">0x38</span><span class="op_lv16">,</span> INTERCEPTION_KEY_DOWN<span class="lv16c">}</span><span class="op_lv0">;</span>
InterceptionKeyStroke del_down  <span class="op_lv0">=</span> <span class="lv16c">{</span><span class="Number">0x53</span><span class="op_lv16">,</span> INTERCEPTION_KEY_DOWN <span class="op_lv16">|</span> INTERCEPTION_KEY_E0<span class="lv16c">}</span><span class="op_lv0">;</span>

<span class="Type">bool</span> <span class="Normal">operator</span> <span class="op_lv0">==</span> <span class="lv16c">(</span><span class="StorageClass">const</span> InterceptionKeyStroke <span class="op_lv16">&amp;</span>first<span class="op_lv16">,</span> <span class="StorageClass">const</span> InterceptionKeyStroke <span class="op_lv16">&amp;</span>second<span class="lv16c">)</span>
<span class="lv16c">{</span>
    <span class="Statement">return</span> first<span class="op_lv16">.</span>code <span class="op_lv16">==</span> second<span class="op_lv16">.</span>code <span class="op_lv16">&amp;&amp;</span> first<span class="op_lv16">.</span>state <span class="op_lv16">==</span> second<span class="op_lv16">.</span>state<span class="op_lv16">;</span>
<span class="lv16c">}</span>

<span class="Type">bool</span> <span class="Normal">operator</span> <span class="op_lv0">!=</span> <span class="lv16c">(</span><span class="StorageClass">const</span> InterceptionKeyStroke <span class="op_lv16">&amp;</span>first<span class="op_lv16">,</span> <span class="StorageClass">const</span> InterceptionKeyStroke <span class="op_lv16">&amp;</span>second<span class="lv16c">)</span>
<span class="lv16c">{</span>
    <span class="Statement">return</span> <span class="op_lv16">!</span><span class="lv15c">(</span>first <span class="op_lv15">==</span> second<span class="lv15c">)</span><span class="op_lv16">;</span>
<span class="lv16c">}</span>

<span class="Type">int</span> main<span class="lv16c">()</span>
<span class="lv16c">{</span>
    <span class="Statement">using</span> <span class="Structure">namespace</span> std<span class="op_lv16">;</span>

    InterceptionContext context<span class="op_lv16">;</span>
    InterceptionDevice device<span class="op_lv16">;</span>
    InterceptionKeyStroke new_stroke<span class="op_lv16">,</span> last_stroke<span class="op_lv16">;</span>

    deque<span class="lv15c">&lt;</span>InterceptionKeyStroke<span class="lv15c">&gt;</span> stroke_sequence<span class="op_lv16">;</span>

    stroke_sequence<span class="op_lv16">.</span>push_back<span class="lv15c">(</span>nothing<span class="lv15c">)</span><span class="op_lv16">;</span>
    stroke_sequence<span class="op_lv16">.</span>push_back<span class="lv15c">(</span>nothing<span class="lv15c">)</span><span class="op_lv16">;</span>
    stroke_sequence<span class="op_lv16">.</span>push_back<span class="lv15c">(</span>nothing<span class="lv15c">)</span><span class="op_lv16">;</span>

    raise_process_priority<span class="lv15c">()</span><span class="op_lv16">;</span>

    context <span class="op_lv16">=</span> interception_create_context<span class="lv15c">()</span><span class="op_lv16">;</span>

    interception_set_filter<span class="lv15c">(</span>context<span class="op_lv15">,</span> interception_is_keyboard<span class="op_lv15">,</span> INTERCEPTION_FILTER_KEY_ALL<span class="lv15c">)</span><span class="op_lv16">;</span>

    <span class="Repeat">while</span><span class="lv15c">(</span>interception_receive<span class="lv14c">(</span>context<span class="op_lv14">,</span> device <span class="op_lv14">=</span> interception_wait<span class="lv13c">(</span>context<span class="lv13c">)</span><span class="op_lv14">,</span> <span class="lv13c">(</span>InterceptionStroke <span class="op_lv13">*</span><span class="lv13c">)</span><span class="op_lv14">&amp;</span>new_stroke<span class="op_lv14">,</span> <span class="Number">1</span><span class="lv14c">)</span> <span class="op_lv15">&gt;</span> <span class="Number">0</span><span class="lv15c">)</span>
    <span class="lv15c">{</span>
        <span class="Conditional">if</span><span class="lv14c">(</span>new_stroke <span class="op_lv14">!=</span> last_stroke<span class="lv14c">)</span>
        <span class="lv14c">{</span>
            stroke_sequence<span class="op_lv14">.</span>pop_front<span class="lv13c">()</span><span class="op_lv14">;</span>
            stroke_sequence<span class="op_lv14">.</span>push_back<span class="lv13c">(</span>new_stroke<span class="lv13c">)</span><span class="op_lv14">;</span>
        <span class="lv14c">}</span>

        <span class="Conditional">if</span><span class="lv14c">(</span>stroke_sequence<span class="lv13c">[</span><span class="Number">0</span><span class="lv13c">]</span> <span class="op_lv14">==</span> ctrl_down <span class="op_lv14">&amp;&amp;</span> stroke_sequence<span class="lv13c">[</span><span class="Number">1</span><span class="lv13c">]</span> <span class="op_lv14">==</span> alt_down <span class="op_lv14">&amp;&amp;</span> stroke_sequence<span class="lv13c">[</span><span class="Number">2</span><span class="lv13c">]</span> <span class="op_lv14">==</span> del_down<span class="lv14c">)</span>
            cout <span class="op_lv15">&lt;&lt;</span> <span class="String">&quot;ctrl-alt-del pressed&quot;</span> <span class="op_lv15">&lt;&lt;</span> endl<span class="op_lv15">;</span>
        <span class="Conditional">else</span> <span class="Conditional">if</span><span class="lv14c">(</span>stroke_sequence<span class="lv13c">[</span><span class="Number">0</span><span class="lv13c">]</span> <span class="op_lv14">==</span> alt_down <span class="op_lv14">&amp;&amp;</span> stroke_sequence<span class="lv13c">[</span><span class="Number">1</span><span class="lv13c">]</span> <span class="op_lv14">==</span> ctrl_down <span class="op_lv14">&amp;&amp;</span> stroke_sequence<span class="lv13c">[</span><span class="Number">2</span><span class="lv13c">]</span> <span class="op_lv14">==</span> del_down<span class="lv14c">)</span>
            cout <span class="op_lv15">&lt;&lt;</span> <span class="String">&quot;alt-ctrl-del pressed&quot;</span> <span class="op_lv15">&lt;&lt;</span> endl<span class="op_lv15">;</span>
        <span class="Conditional">else</span>
            interception_send<span class="lv14c">(</span>context<span class="op_lv14">,</span> device<span class="op_lv14">,</span> <span class="lv13c">(</span>InterceptionStroke <span class="op_lv13">*</span><span class="lv13c">)</span><span class="op_lv14">&amp;</span>new_stroke<span class="op_lv14">,</span> <span class="Number">1</span><span class="lv14c">)</span><span class="op_lv15">;</span>

        <span class="Conditional">if</span><span class="lv14c">(</span>new_stroke<span class="op_lv14">.</span>code <span class="op_lv14">==</span> SCANCODE_ESC<span class="lv14c">)</span> <span class="Statement">break</span><span class="op_lv15">;</span>

        last_stroke <span class="op_lv15">=</span> new_stroke<span class="op_lv15">;</span>
    <span class="lv15c">}</span>

    interception_destroy_context<span class="lv15c">(</span>context<span class="lv15c">)</span><span class="op_lv16">;</span>

    <span class="Statement">return</span> <span class="Number">0</span><span class="op_lv16">;</span>
<span class="lv16c">}</span>
</pre>

The following sample identify multiple mice through left clicking (worked well for my magic mouse and touch pad):

<pre>
<span class="Include">#include </span><span class="String">&lt;interception.h&gt;</span>
<span class="Include">#include </span><span class="String">&quot;utils.h&quot;</span>

<span class="Include">#include </span><span class="String">&lt;iostream&gt;</span>

<span class="Structure">enum</span> ScanCode
<span class="lv16c">{</span>
    SCANCODE_ESC <span class="op_lv16">=</span> <span class="Number">0x01</span>
<span class="lv16c">}</span><span class="op_lv0">;</span>

<span class="Type">int</span> main<span class="lv16c">()</span>
<span class="lv16c">{</span>
    <span class="Statement">using</span> <span class="Structure">namespace</span> std<span class="op_lv16">;</span>

    InterceptionContext context<span class="op_lv16">;</span>
    InterceptionDevice device<span class="op_lv16">;</span>
    InterceptionStroke stroke<span class="op_lv16">;</span>

    raise_process_priority<span class="lv15c">()</span><span class="op_lv16">;</span>

    context <span class="op_lv16">=</span> interception_create_context<span class="lv15c">()</span><span class="op_lv16">;</span>

    interception_set_filter<span class="lv15c">(</span>context<span class="op_lv15">,</span> interception_is_keyboard<span class="op_lv15">,</span> INTERCEPTION_FILTER_KEY_DOWN <span class="op_lv15">|</span> INTERCEPTION_FILTER_KEY_UP<span class="lv15c">)</span><span class="op_lv16">;</span>
    interception_set_filter<span class="lv15c">(</span>context<span class="op_lv15">,</span> interception_is_mouse<span class="op_lv15">,</span> INTERCEPTION_FILTER_MOUSE_LEFT_BUTTON_DOWN<span class="lv15c">)</span><span class="op_lv16">;</span>

    <span class="Repeat">while</span><span class="lv15c">(</span>interception_receive<span class="lv14c">(</span>context<span class="op_lv14">,</span> device <span class="op_lv14">=</span> interception_wait<span class="lv13c">(</span>context<span class="lv13c">)</span><span class="op_lv14">,</span> <span class="op_lv14">&amp;</span>stroke<span class="op_lv14">,</span> <span class="Number">1</span><span class="lv14c">)</span> <span class="op_lv15">&gt;</span> <span class="Number">0</span><span class="lv15c">)</span>
    <span class="lv15c">{</span>
        <span class="Conditional">if</span><span class="lv14c">(</span>interception_is_keyboard<span class="lv13c">(</span>device<span class="lv13c">)</span><span class="lv14c">)</span>
        <span class="lv14c">{</span>
            InterceptionKeyStroke <span class="op_lv14">&amp;</span>keystroke <span class="op_lv14">=</span> <span class="op_lv14">*</span><span class="lv13c">(</span>InterceptionKeyStroke <span class="op_lv13">*</span><span class="lv13c">)</span> <span class="op_lv14">&amp;</span>stroke<span class="op_lv14">;</span>

            <span class="Conditional">if</span><span class="lv13c">(</span>keystroke<span class="op_lv13">.</span>code <span class="op_lv13">==</span> SCANCODE_ESC<span class="lv13c">)</span> <span class="Statement">break</span><span class="op_lv14">;</span>
        <span class="lv14c">}</span>

        <span class="Conditional">if</span><span class="lv14c">(</span>interception_is_mouse<span class="lv13c">(</span>device<span class="lv13c">)</span><span class="lv14c">)</span>
        <span class="lv14c">{</span>
            InterceptionMouseStroke <span class="op_lv14">&amp;</span>mousestroke <span class="op_lv14">=</span> <span class="op_lv14">*</span><span class="lv13c">(</span>InterceptionMouseStroke <span class="op_lv13">*</span><span class="lv13c">)</span> <span class="op_lv14">&amp;</span>stroke<span class="op_lv14">;</span>

            cout <span class="op_lv14">&lt;&lt;</span> <span class="String">&quot;INTERCEPTION_MOUSE(&quot;</span> <span class="op_lv14">&lt;&lt;</span> device <span class="op_lv14">-</span> INTERCEPTION_MOUSE<span class="lv13c">(</span><span class="Number">0</span><span class="lv13c">)</span> <span class="op_lv14">&lt;&lt;</span> <span class="String">&quot;)&quot;</span> <span class="op_lv14">&lt;&lt;</span> endl<span class="op_lv14">;</span>
        <span class="lv14c">}</span>

        interception_send<span class="lv14c">(</span>context<span class="op_lv14">,</span> device<span class="op_lv14">,</span> <span class="op_lv14">&amp;</span>stroke<span class="op_lv14">,</span> <span class="Number">1</span><span class="lv14c">)</span><span class="op_lv15">;</span>
    <span class="lv15c">}</span>

    interception_destroy_context<span class="lv15c">(</span>context<span class="lv15c">)</span><span class="op_lv16">;</span>

    <span class="Statement">return</span> <span class="Number">0</span><span class="op_lv16">;</span>
<span class="lv16c">}</span>
</pre>                                                                                                           

As you can see interception\_is\_keyboard and interception\_is\_mouse are convenience device selection predicates, but you can implement your own.

The following sample allows one to query for a device’s "hardware id", which may help on disambiguation of device input.  
Just remember this hardware id’s are not required to be unique, but mostly will when you have at last two different device models.

<pre>
<span class="Include">#include </span><span class="String">&lt;interception.h&gt;</span>
<span class="Include">#include </span><span class="String">&quot;utils.h&quot;</span>

<span class="Include">#include </span><span class="String">&lt;iostream&gt;</span>

<span class="Structure">enum</span> ScanCode
<span class="lv16c">{</span>
    SCANCODE_ESC <span class="op_lv16">=</span> <span class="Number">0x01</span>
<span class="lv16c">}</span><span class="op_lv0">;</span>

<span class="Type">int</span> main<span class="lv16c">()</span>
<span class="lv16c">{</span>
    <span class="Statement">using</span> <span class="Structure">namespace</span> std<span class="op_lv16">;</span>

    InterceptionContext context<span class="op_lv16">;</span>
    InterceptionDevice device<span class="op_lv16">;</span>
    InterceptionStroke stroke<span class="op_lv16">;</span>

    <span class="Type">wchar_t</span> hardware_id<span class="lv15c">[</span><span class="Number">500</span><span class="lv15c">]</span><span class="op_lv16">;</span>

    raise_process_priority<span class="lv15c">()</span><span class="op_lv16">;</span>

    context <span class="op_lv16">=</span> interception_create_context<span class="lv15c">()</span><span class="op_lv16">;</span>

    interception_set_filter<span class="lv15c">(</span>context<span class="op_lv15">,</span> interception_is_keyboard<span class="op_lv15">,</span> INTERCEPTION_FILTER_KEY_DOWN <span class="op_lv15">|</span> INTERCEPTION_FILTER_KEY_UP<span class="lv15c">)</span><span class="op_lv16">;</span>
    interception_set_filter<span class="lv15c">(</span>context<span class="op_lv15">,</span> interception_is_mouse<span class="op_lv15">,</span> INTERCEPTION_FILTER_MOUSE_LEFT_BUTTON_DOWN<span class="lv15c">)</span><span class="op_lv16">;</span>

    <span class="Repeat">while</span><span class="lv15c">(</span>interception_receive<span class="lv14c">(</span>context<span class="op_lv14">,</span> device <span class="op_lv14">=</span> interception_wait<span class="lv13c">(</span>context<span class="lv13c">)</span><span class="op_lv14">,</span> <span class="op_lv14">&amp;</span>stroke<span class="op_lv14">,</span> <span class="Number">1</span><span class="lv14c">)</span> <span class="op_lv15">&gt;</span> <span class="Number">0</span><span class="lv15c">)</span>
    <span class="lv15c">{</span>
        <span class="Conditional">if</span><span class="lv14c">(</span>interception_is_keyboard<span class="lv13c">(</span>device<span class="lv13c">)</span><span class="lv14c">)</span>
        <span class="lv14c">{</span>
            InterceptionKeyStroke <span class="op_lv14">&amp;</span>keystroke <span class="op_lv14">=</span> <span class="op_lv14">*</span><span class="lv13c">(</span>InterceptionKeyStroke <span class="op_lv13">*</span><span class="lv13c">)</span> <span class="op_lv14">&amp;</span>stroke<span class="op_lv14">;</span>

            <span class="Conditional">if</span><span class="lv13c">(</span>keystroke<span class="op_lv13">.</span>code <span class="op_lv13">==</span> SCANCODE_ESC<span class="lv13c">)</span> <span class="Statement">break</span><span class="op_lv14">;</span>
        <span class="lv14c">}</span>

        <span class="Type">size_t</span> length <span class="op_lv15">=</span> interception_get_hardware_id<span class="lv14c">(</span>context<span class="op_lv14">,</span> device<span class="op_lv14">,</span> hardware_id<span class="op_lv14">,</span> <span class="Normal">sizeof</span><span class="lv13c">(</span>hardware_id<span class="lv13c">)</span><span class="lv14c">)</span><span class="op_lv15">;</span>

        <span class="Conditional">if</span><span class="lv14c">(</span>length <span class="op_lv14">&gt;</span> <span class="Number">0</span> <span class="op_lv14">&amp;&amp;</span> length <span class="op_lv14">&lt;</span> <span class="Normal">sizeof</span><span class="lv13c">(</span>hardware_id<span class="lv13c">)</span><span class="lv14c">)</span>
            wcout <span class="op_lv15">&lt;&lt;</span> hardware_id <span class="op_lv15">&lt;&lt;</span> endl<span class="op_lv15">;</span>

        interception_send<span class="lv14c">(</span>context<span class="op_lv14">,</span> device<span class="op_lv14">,</span> <span class="op_lv14">&amp;</span>stroke<span class="op_lv14">,</span> <span class="Number">1</span><span class="lv14c">)</span><span class="op_lv15">;</span>
    <span class="lv15c">}</span>

    interception_destroy_context<span class="lv15c">(</span>context<span class="lv15c">)</span><span class="op_lv16">;</span>

    <span class="Statement">return</span> <span class="Number">0</span><span class="op_lv16">;</span>
<span class="lv16c">}</span>
</pre>
