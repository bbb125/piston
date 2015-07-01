# **WARNING** #
I created piston and this debugger to accomplish what `*i*` needed.

Although I tried to make it generic and potentially usable by others, there are some ugly bits and not very much polish. I didn't really intend to release this when I coded it. You have been warned.


---

# Introduction #

This page details the protocol used to communicate between the eclipse debugger and the javascript app.

When I refer to 'debugger' I'm talking about the Eclipse Debugger Plugin.
When I refer to 'piston' I refer to the javascript app (pistonmonkey in this projects case)


---

# Overview #

It starts in Eclipse:
  1. Menu: Run->Debug Configurations...
  1. Right click on 'Piston Compatible App' on the left and choose 'New'
  1. Give it a name
  1. Browse to where the piston binary is at
  1. Browse to the JavaScript file you want to start debugging in is at

When you hit 'Debug' it will launch the piston app as:

pistonmonkey --debugMode --debugHost <piston host> --debugPort <piston port> <js file>

At this point the debugger waits for piston to start up.
Piston will start up and will listen on the host and port that it was instructed to when launched. (default 'localhost:7570')

Once the debugger see's that piston is listening on 7570 the debugger will continue starting up.
The debugger will tell piston which breakpoints are already set.
It will then tell piston what the host and port is that the debugger itself will be listening on (default 'localhost:7580')
The debugger then tells piston that it's done and debugging can begin (start).

piston then executes the javascript file it was told to execute.
When any breakpoints are encountered it will let the debugger know.
Likewise when a breakpoint is reached and you hit step over, or continue, etc. in the debugger, the debugger will send those messages to piston.


---

# Details #

The protocol the two use to communicate is very basic. It's somewhat REST like.

Each 'message' has a name and a set of values which are POST'ed to a URL.
http://localhost:7570/messageName

Although we use a 'POST' the actual content isn't form URL encoded right now.
It's just a bunch of \n seperated values (with a trailing \n)

| **Message Name** | **Sent From** | **Data** | **Description** |
|:-----------------|:--------------|:---------|:----------------|
| breakpoint\_set  | debugger      | `<filename>\n<line num>\n` | Sent whenever a breakpoint is set in the debugger. When the debugger first starts it will send one of these for each pre-existing breakpoint before launching |
| debugger\_host   | debugger      | `<host>\n` | Sent to piston to inform it what host the debugger will be listening on. Piston will send messages to this host after it receives start |
| debugger\_port   | debugger      | `<port>\n` | Sent to piston to inform it what port the debugger will be listening on. Piston will send messages to this port after it receives start |
| start            | debugger      |          | Sent when the debugger is done setting initial breakpoints and wants piston to start executing javascript |
| thread\_created  | piston        | `<thread name>\n<thread num>\n` | Sent whenever a new thread is created in piston. This maps to a JSContext in spidermonkey. You must send at least one of these for the 'Main' thread. The thread num should increment for each new thread, starting at zero |
| breakpoint\_reached | piston        | `<thread num>\n<filename>\n<line num>\n` | Sent to the debugger whenever a breakpoint is reached. Execution is paused in piston until the debugger sends a command in the future to step or continue. The filename should be a full path that the debugger can resolve regardless of working directory. Both sides should use innodes to compare files rather than paths due to symlinks, etc. |
| stack\_get       | debugger      | `<thread num>\n` | This is sent after a breakpoint\_reached message is received in order to get the call stack for display in the debugger. Piston responds with data in the response. Each line is an item in the call stack. The format of the line is `<file name>:<line num>\n` |
| variables\_get   | debugger      | `<thread num>\n` | This is sent by the debugger to get a list of variables currently visible to the thread where we are breakpointed at. Piston responds with data in the response. Each line is a different variable. The format of each line is `<variable name>\t<variable type>\t<has children>\t<base64 encoded variable value>\n` The `<has children>` param is a boolean (true or false) and right now only false is supported. |
| continue         | debugger      | `<thread num>\n` | Sent by the debugger when the user hits the 'Play' or 'Continue' button to resume execution. |
| step\_into       | debugger      | `<thread num>\n` | Sent by the debugger when the user hits the 'Step Into' button. Piston should now resume execution and when the execution has proceeded 1 step into should send the 'stepped' command back. |
| step\_over       | debugger      | `<thread num>\n` | Sent by the debugger when the user hits the 'Step Over' button. Piston should now resume execution and when the execution has proceeded 1 step over should send the 'stepped' command back. |
| stepped          | piston        | `<thread num>\n<filename>\n<line num>\n` | Sent back once piston has reached the final destination of the last step command. The filename and line num should be where we are currently at now. Again, execution is now paused in piston until the debugger tells it what to do next |
| terminated       | piston        |          | Sent by piston when the process is done and is about to exit. |

As you can see from above, this is about as basic as you can get while still supporting enough stuff to be useful. Notice that things like step return, child variables and watchpoints are not yet supported by the current code base.

It would be pretty straightforward for someone to swap out pistonmonkey with another javascript engine, or swap out the eclipse debugger with something else like a command line one.