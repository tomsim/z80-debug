# Design

## Overview

~~~
                                                          ┌───┐
                                                          │   │
┌───────────────────┐              ┌─────────────────┐    │ S │   ┌────────────────────┐
│                   │   Request    │                 │    │ o │   │Emulator            │
│                   │─────────────▶│       z80       │────┼─c─┼──▶│   ┌────────────────┴───┐
│      vscode       │              │      debug      │    │ k │   │   │MAME                │
│                   │◀─────────────│     adapter     │◀───┼─e─┼───│   │   ┌────────────────┴──┐
│                   │  Response    │                 │    │ t │   │   │   │ZEsarUX            │
└───────────────────┘              └─────────────────┘    │   │   └───┤   │                   │
                                                          └───┘       │   │                   │
                                                                      └───┤                   │
                                                                          │                   │
                                                                          └───────────────────┘
~~~~

## Main Classes


- DebugAdapter: Just runs Z80Debug.
- Extension: The extension class. Activates the extension and registers commands.
- Frame: Represents a Z80 StackObject, i.e. caller address and objects on stack.
- Labels: Singleton which reads the .list and .labels file and associates addresses, labels, filenames and line numbers.
- Settings: Singleton to hold the extension's settings.
- ShallowVar: DisassemblyVar, RegistersMainVar, RegistersSecondaryVar, StackVar, LabelVar. Representations of variables. They know how to retrieve the data from zesarux.
- Z80Registers: Static class to parse (from zesarux) and format registers.
- StateZ80: Class to get and set the complete machine state.
- Emulator: Gets requests from vscode and passes them to zesarux (via ZesaruxSocket).
- ZesaruxSocket: Socket connection and communication to zesarux emulator. Knows about the basic communication, but not the commands.


Helper classes:
- CallSerializer: Used to queue/serialize function calls from vscode and to the socket.
- Log: Used for logging. Adds the caller and break time to the logs.
- RefList: List to associate variable references to objects.

~~~
┌─────────┐      ┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐     ┌─────────────────┐
│         │      │                                                                                                     │     │Helper           │
│         │      │                                            DebugAdapter                                             │     │                 │
│         │      │                                                                                                     │     │                 │
│         │      └─────────────────────────────────────────────────────────────────────────────────────────────────────┘     │┌─────────┐      │
│         │         ▲                         ▲                          ▲                                 ▲                 ││Utility  │      │
│         │         │                         │                          │                                 │                 │└─────────┘      │
│Settings │         ▼                         │                          ▼                                 ▼                 │                 │
│         │    ┌───────────────┐              ▼               ┌────────────────────┐             ┌──────────────────────┐    │┌──────────────┐ │
│         │    │TextView       │         ┌─────────┐          │Emulator            │             │Variables             │    ││CallSerializer│ │
│         │    │ ┌─────────────┴─┐       │         │          │  ┌─────────────────┴──┐          │┌───────────┐         │    │└──────────────┘ │
│         │    │ │MemoryDumpView │       │ Labels  │◀────────▶│  │ZesaruxEmulator     │          ││ShallowVar │         │    │                 │
│         │    │ │ ┌─────────────┴─┐     │         │  ┌──────▶│  │  ┌─────────────────┴──┐       │└───────────┘         │    │┌─────────┐      │
│         │    │ │ │MemoryReg.View │     └─────────┘  │       │  │  │ZesaruxExtEmulator  │       │┌────────────────┐    │    ││RefList  │      │
└─────────┘    │ │ │ ┌─────────────┴─┐        ▲       │       │  │  │  ┌─────────────────┴──┐    ││DisassemblyVar  │    │    │└─────────┘      │
               └─┤ │ │ZxN.SpritesView│        │       │       └──┤  │  │MameEmulator        │    │└────────────────┘    │    │                 │
                 │ │ │               │        ▼       │          │  │  │                    │    │┌────────────────┐    │    │┌────┐           │
                 └─┤ │               │  ┌ ─ ─ ─ ─ ─ ┐ │          └──┤  │                    │    ││RegisterMainVar │    │    ││Log │           │
                   │ │               │      Files     │             │  │                    │    │└────────────────┘    │    │└────┘           │
                   └─┤               │  └ ─ ─ ─ ─ ─ ┘ │             └──┤                    │    │┌────────────────────┐│    │                 │
                     │               │                │            ▲   │                    │    ││RegisterSecondaryVar││    │                 │
                     └───────────────┘                │            │   └────────────────────┘    │└────────────────────┘│    │                 │
                        ▲                             │            ▼            ▲        ▲       │┌──────────┐          │    │                 │
                        │                             │  ┌──────────────┐       │        │       ││StackVar  │          │    │                 │
                        └─────────────────────────────┘  │              │       ▼        │       │└──────────┘          │    │                 │
                                                         │ Z80Registers │  ┌──────────┐  │       │┌──────────┐          │    │                 │
                                                         │              │  │ StateZ80 │  │       ││LabelVar  │          │    │                 │
                                                         └──────────────┘  └──────────┘  │       │└──────────┘          │    │                 │
                                                                   ▲            ▲        │       └──────────────────────┘    └─────────────────┘
                                                                   │            │        │                  ▲
                                                                   │            │        │                  │
                                                                   ▼            ▼        ▼                  ▼
                                                              ┌─────────────────────────────────────────────────────────┐
                                                              │Sockets ┌──────────────┐       ┌───────────────┐         │
                                                              │        │  MameSocket  │       │ ZesaruxSocket │         │
                                                              │        └──────────────┘       └───────────────┘         │
                                                              └─────────────────────────────────────────────────────────┘
~~~~


Communication:

DebugAdapter <-> Emulator:
DebugAdapter takes care of association of vscode references and objects.
- Commands: step-over, step-into, step-out
- Data: Frames (call stack), Registers/Frame, expressions, breakpoints, Labels.

Emulator <-> Socket:
- Commands: run, step-over, step-into, step-out, breakpoints
- Data: registers, memory dump, call stack



## Asynchronicity

vscode is highly asynchronous. All requests start with the 'request' and end with a 'response'. The 'response' would be typically generated in another function e.g. as a response from the zesarux socket answer.
Meanwhile vscode could have sent another request either for the same
(see stackTraceRequest) or for something else.

Since this debug adapter has to maintain a reference/object map/list it is
necessary to reset this list sometimes (to free the objects).
This list is global whcih leads to asynchronicity problems.

Therefore all requests (next, scopesRequest, stackTraceRequest, variablesRequest etc.) from vscode are added to a queue the so-called
CallSerializer (this.serializer). So that they are executed and responded to in exactly the order they come in.


## Showing variables, call stacks etc.

Whenever vscode thinks it needs to update some of the areas (VARIABLES, WATCHES, CALL STACK) it does a request.
The request might be chained. E.g. if an entry in the CALL STACK is selected this first requests the SCOPE for the call stack (frame) ID.
The scopes are displayed as header lines in the VARIABLES area.
They are very similar to variables but appear as head line.
In this adapter they are used for
- The disassembly (either starting at the PC or if a different callstack is selected the disassembly starts at the call stack address).
- The Z80 registers (divided in main and secondary registers).
- The stack, i.e. the stack that is currently used for push/pop.

For each scope a variables request is done. Here a variable is returned for each shown line. E.g. one line/variable for each register or one line/variable for each disassembly line.


vscode works with IDs/numbers only. Unfortunately it is not possible
to response with an array of objects that vscode passes back in
the next request.
Instead it is necessary to work with IDs/numbers.

So, when there is a stackTraceRequest (call stack) a list is used to map the IDs to objects.
At the start of the request the list is cleared. Then, for every returned frame (call stack entry) a new ID is generated (the index to the list is simply increased).
The object is created. It holds all necessary data for processing the next step. And the object is put in the list.

Here are the requests and the list/objects involved:

### stackTraceRequest:

List:
listFrames, cleared on start of request, filled until response.
listVariables, cleared on start of request, filled by other requests.

Object:
- address: the address in the call stack (required for disassembly)
- fileName, lineNr: file and line number so that vscode can show the corresponding file when selected.


### scopesRequest:

The returned scopes list is basically always the same:
'Disassembly', 'Registers', 'Registers 2', 'Stack'
However, for each scope a variable reference is passed. This reference ID is different for different frames.
I.e. an object is created that includes necessary information for decoding in the variablesRequest.

'Disassembly':
- address

'Registers'/'Registers 2':
- none

'Stack':
- address
- size

List:
listVariables, filled until response.



### variablesRequest:
List:
listVariables, is only read.

### evaluateRequest:

An expression is past for hovering or 'add watch'. It normally
contains a label for wich a shallow var is constructed.

'Labels':
- pseudo variables for
	- byte array
	- word array

'Memory Dump' (byte/word array):
- address


### Shallow Variable References

A variable that has not been requested does exist only as number (ID) with associated shallow object.
The shallow object contains enough information to actually retrieve the variable information.
For variables a class is created that behaves similar for all different types of variables.

Basically it includes a anonymous function that has the knowledge to retrieve the data.

Examples:
- E.g. the call stack variable gets a function that communicates with the socket to retrieve the stack values.
- From evaluate (e.g. 'add watch') a label has been added. This is immediately displayed. The responses contained a var ID corresponding to an object with a function that communicates with the socket to retrieve a memory dump for that address.
If the variable is opened in the WATCH area a variable request is done for that ID. The corresponding object's function is executed and the data is retrieved and returned in the response.


# Problems / Decisions needed

vscode is highly asynchronous. All requests start with the 'request' and end with a 'response'. The 'response' would be typically generated in another function e.g. as a response from the zesarux socket answer.
Meanwhile vscode could have sent another request either for the same
(see stackTraceRequest) or for something else.
This becomes even more complicated because variables (displayed in vscode)
are referenced by a number only.
Therefore the debug adapter needs to keep track with a map or list of the variable references and the underlying objects.
Unfortunately there is no information available from vscode whenever a variable reference is not required anymore.
I.e. it is impossible to delete variable references and the underlying objects.
Because of the asynchronicity it is also imposible to bound the deleting
of objects to certain requests (like the stack trace request).

It would be much easier if vscode (instead of using reference numbers) would offer the possibility to set a variables reference as an object. Then simply garbage collection would take care of unused objects.

A way out would be to use a layer in between (between vscode and the emulator/socket).
This layer would request all required data wenever a 'next' (step) has been performed.
So all data is allocated for one 'step' only. At the start of the 'step' the old data is discarded and new data structures are allocated.
vscode requests would be ansewered (immediately) just by returning the already received data.

Problem with this approach is that this layer would have to requests all data at once and for every step.
E.g. even while stepping from one assembler instruction to the next all disassemblys (and, when this available later, all registers/stack objects
for that call stack item) need to be retrieved from the emulator in advance.
Even if the data in the vscode UI is collapsed.

This doesn't seem the rigth way. Let's see how the discussion with vscode guys will turn out.


# Other Components

## MemoryDumpView Sequence Diagrams

```puml
hide footbox
title Step
actor User
User -> vscode: Step
vscode -> EmulDebugAdapter: stepXxxRequest
EmulDebugAdapter -> MemoryDumpView: update
MemoryDumpView -> Emulator: getMemoryDump(s)
MemoryDumpView <-- Emulator: Data
note over MemoryDumpView: create html+js
MemoryDumpView -> webView: Set webview.html
```

```puml
hide footbox
title Hover
actor User
User -> webView: Hover
webView -> MemoryDumpView: getValueInfoText/getAddressInfoText
webView <- MemoryDumpView: valueInfoText/addressInfoText
```

```puml
hide footbox
title Change Memory Value
actor User
User -> webView: DoubleClick
webView -> MemoryDumpView: valueChanged
MemoryDumpView -> MemoryDumpView: changeMemory
MemoryDumpView -> Emulator: writeMemory
webView <- MemoryDumpView: changeValue
```


## ZXNextSpritesView Sequence Diagrams

```puml
hide footbox
title Step
actor User
User -> vscode: Step
vscode -> EmulDebugAdapter: stepXxxRequest
EmulDebugAdapter -> ZXNextSpritesView: update
ZXNextSpritesView -> ZXNextSpritesView: getSprites
ZXNextSpritesView -> Emulator: getTbblueSprite(s)
ZXNextSpritesView <-- Emulator: Data
alt palette empty
    ZXNextSpritesView -> ZXNextSpritesView: getSpritePalette
    ZXNextSpritesView -> Emulator: getTbblueRegister(whichPaletteXXX)
    ZXNextSpritesView <-- Emulator: Used palette
    ZXNextSpritesView -> Emulator: getTbbluePalette
    ZXNextSpritesView <-- Emulator: Data
end
alt patterns empty
    ZXNextSpritesView -> ZXNextSpritesView: getSpritePatterns
    ZXNextSpritesView -> Emulator: getTbbluePattern(s)
    ZXNextSpritesView <-- Emulator: Data
end
note over ZXNextSpritesView: create html+js
ZXNextSpritesView -> webView: Set webview.html
```

```puml
hide footbox
title Reload Patterns
actor User
User -> webView: Click "Reload"

webView -> ZXNextSpritesView: getSpritePalette
    ZXNextSpritesView -> Emulator: getTbblueRegister(whichPaletteXXX)
    ZXNextSpritesView <-- Emulator: Used palette
    ZXNextSpritesView -> Emulator: getTbbluePalette
    ZXNextSpritesView <-- Emulator: Data

webView -> ZXNextSpritesView: getSpritePatterns
    ZXNextSpritesView -> Emulator: getTbbluePattern(s)
    ZXNextSpritesView <-- Emulator: Data

note over ZXNextSpritesView: create html+js
ZXNextSpritesView -> webView: Set webview.html
```

