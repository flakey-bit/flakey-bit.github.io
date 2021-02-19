---
id: 2124
title: Using .NET to interact with the Chrome V8 debugger (running in ClearScript)
date: 2019-05-26T19:22:05+10:00
author: eddiewould
layout: post
guid: https://eddiewould.com/?p=2124
permalink: /2019/05/26/using-net-to-interact-with-the-chrome-v8-debugger-running-in-clearscript/
ocean_gallery_link_images:
  - 'off'
ocean_sidebar:
  - "0"
ocean_second_sidebar:
  - "0"
ocean_disable_margins:
  - enable
ocean_display_top_bar:
  - default
ocean_display_header:
  - default
ocean_center_header_left_menu:
  - "0"
ocean_custom_header_template:
  - "0"
ocean_header_custom_menu:
  - "0"
ocean_menu_typo_font_family:
  - "0"
ocean_disable_title:
  - default
ocean_disable_heading:
  - default
ocean_disable_breadcrumbs:
  - default
ocean_display_footer_widgets:
  - default
ocean_display_footer_bottom:
  - default
ocean_custom_footer_template:
  - "0"
ocean_link_format_target:
  - self
ocean_quote_format_link:
  - post
categories:
  - Uncategorized
tags:
  - V8 debugger debugging ClearScript .NET JavaScript
---

Exploring the Google Chrome debugger protocol by trying to talk to ClearScript running in V8.

An enterprise app I develop & maintain makes use of <a href="https://github.com/microsoft/ClearScript">ClearScript</a> as an extensibility point: end-users (semi-technical) write financial algorithms in JavaScript and those algorithms are evaluated by the system during batch processing (certain key C# types are exposed that the algorithms can make use of). ClearScript is set up to run the JavaScript against the Chrome "V8" JavaScript engine (the same as NodeJS and, well, Chrome).

Some of these algorithms are quite complex and time consuming to write - the question arose "Can we provide a way for end-users to debug the algorithms"?

As a minimum-viable-product I decided users would need the ability to

* Set breakpoints
* Inspect the value of variables
* Step over/into code
* Modify the value of variables in-place (nice-to-have)

As noted my StackOverflow <a href="https://stackoverflow.com/questions/56207320">question</a>, whilst I'm (as a developer) able to connect to the V8 engine w/ Chrome DevTools, that's not ideal for end-users for a couple of reasons:

* Tricky to set up / not integrated with the rest of the web app
* Probably won't play nicely with corporate firewall (every simultaneous instance of the debugger will have to listen on a unique port; That's another port we'd need to let through the firewall).

I was therefore wondering - is there a .NET interface to the Chrome V8 debugger? *I didn't find one* so set out about building one. 

SIDE NOTE: After building mine, I found <a href="https://github.com/BaristaLabs/chrome-dev-tools-runtime">chrome-dev-tools-runtime</a> - it looks pretty similar to what I implemented (but more/less elegant in places). Note that as well as a concrete implementation, the same author (BaristaLabs) offers a <a href="https://github.com/BaristaLabs/chrome-dev-tools-generator">generator</a> for creating DTOs for the various messages/events for the protocol (the protocol is constantly evolving as Chrome evolves). That should be less of an issue when targetting ClearScript though.

Even after reading what I could find in terms of official documtation <a href="https://chromedevtools.github.io/devtools-protocol/">here</a> & <a href="https://github.com/ChromeDevTools/awesome-chrome-devtools#chrome-devtools-protocol">here</a> I still didn't really understand how the protocol works. With a bit of <a href="https://www.google.com/search?q=fiddler+js&rlz=1C1CHBF_enAU752AU752&oq=fiddler+js&aqs=chrome..69i57j69i59j69i60l2j69i59j69i61.1995j0j7&sourceid=chrome&ie=UTF-8#">Fiddler</a> + <a href="https://www.wireshark.org">Wireshark</a> and general prodding I managed to talk to it. Here's what I found:

* It's primarily a websocket based protocol (asynchronous JSON messages). That said, *the initial setup of the session needs to be done through HTTP* (I'm not referring to the websocket protocol escalation).
* Client code can interact with the debugger by sending commands & (asyncronously) waiting for a response (commands have an `id` property which is used a sequence number to correlate requests and responses).
* The browser/debugger also publishes events to the same websocket - that slightly complicates processing of messages (since any given message could either be a response to a command _or_ an event)

In the case of ClearScript hosted V8, the initial setup was as simple as making a `GET` to `http://{hostname}:{port}/json/list` - that returns a list of open "tabs". In a ClearScript scenario (as opposed to real Chrome) I think we'll only ever have one tab. The tab DTO has an `id` property - we need the `id` to setup the WebSocket:


```csharp
await _socket.ConnectAsync(new Uri($"ws://{_hostname}:{_port}/{tabInfo.Id}"), cancellationToken);
```

Issuing a command is as simple as serializing the command to JSON and sending it to the websocket. The websocket should be read periodically for command responses & events (also JSON).

Other miscellaneous information:

* When ClearScript evaluates a script, a `debugger` statement won't do anything unless there's a debugger attached (just like real Chrome needs DevTools open). Use the flags `EnableDebugging` and `AwaitDebuggerAndPauseOnStart`

```csharp
V8ScriptEngineFlags flags = V8ScriptEngineFlags.EnableDebugging |
                              V8ScriptEngineFlags.AwaitDebuggerAndPauseOnStart;
new V8ScriptEngine(flags);
```

* A client connecting to the debugger (via websocket) is not enough to satisfy `AwaitDebuggerAndPauseOnStart` - the client must issue the commands `Debugger.enable` &  `Runtime.runIfWaitingForDebugger` - after that point, even if the client disconnects the script will continue to completion.
* There is an experimental feature in Chrome Devtools whereby you can get <a href="https://chromedevtools.github.io/devtools-protocol/">DevTools to show the debugger protocol commands</a> it is executing (see screenshot below). Extremely handy to know about!

<figure class="wp-block-image"><img src="/wp-content/uploads/2019/05/image-1024x699.png" alt="" class="wp-image-2131"/></figure>

Some useful commands:
* `Runtime.enable`
* `Debugger.enable`
* `Runtime.runIfWaitingForDebugger`
* `Debugger.getScriptSource`
* `Debugger.resume`
* `Debugger.evaluateOnCallFrame`
* `Runtime.getProperties`

I've built a `Communicator` class to abstract the V8 debugger interactions. The interface looks as follows:

```csharp
public interface ICommunicator
{
    Task Connect(CancellationToken cancellationToken);
    Task<string> SendCommand(string method, object parameters = null);
    Task<TEvent> WaitForEventAsync<TEvent>(CancellationToken token) where TEvent : IV8EventParameters;
}
```

Consuming could would instantiate the communicator (in a `using` block), `Connect` to it, invoke `SendCommand` several times, possibly wait on the `DebuggerPaused` / `ScriptParsed` events and finally `Disconnect`.

The implementation of `Communicator` makes use of `MessagePump` (a task running on a separate thread to poll & read messages (both events and responses to commands) from the websocket. The `MessagePump` raises events which the `Communicator` handles. 

Things get a little bit fruity in the `Communicator` class - it creates three <a href="https://docs.microsoft.com/en-us/dotnet/api/system.threading.channels.channel?view=dotnet-plat-ext-2.1">channels</a> (one for each of the event types + another for command responses) and adds event handlers that write to the appropriate channel. The implementation of `WaitForDebuggerPausedEventAsync` and `WaitForScriptParsedEventAsync`is trivial - we just read asyncronously from the appropriate channel. Obviously, *this approach wouldn't scale* if we were interested in multiple types of events. Calling could would probably like a generic `WaitForEventAsync<TEvent>` method anyway. I'm not sure how you'd achieve that.

*UPDATE*: 3rd September 2019: I've refactored things to remove the channels from the `Communicator` class - instead, the code now maintains a dictionary of buffered events as well as a dictionary of task completion sources. When an event is fired or awaited we will pair of the first matching task completion source with the (possibly buffered) event.

A sample solution (showing a ClearScript target script being debugged by another process) is up on <a href="https://github.com/flakey-bit/ClearScriptDebugging">Github</a> if you're interested.

## Conclusion

Whilst I was able to achieve the goals I set out to (set breakpoint, step over/into, examine/modify variables) there's clearly a lot of moving parts - integrating it into the production app would require careful management of threads to ensure things don't get stuck / resources exhausted. 

Hope you enjoyed the post!