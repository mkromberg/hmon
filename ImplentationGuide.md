# Implementation guide for a HMon Client and Application

This document is a standalone documentation for what is needed to make use of Dyalogs' Health Monitor protocol. However, this document takes offspring in the Demo application developed by Morten Krumberg and any code examples will be referencing this.

This document will not go into detail about the protocol used, instead we refer to the official documentation for the protocol, which can be found [here](https://github.com/Dyalog/HMon/blob/main/docs/HMON.md). 

It is worth mentioning that the code example has reversed the roles of the Client and Application it references to the documentation. 

<table>
<tr>
    <th>Expression</th>
    <th>Meaning</th>
</tr>
<tr>
    <td>Client</td>
    <td>Health Monitorer</td>
</tr>
<tr>
    <td>Application</td>
    <td>The interpreter running the program of interest</td>
</tr>
</table>

## Configuring the Application

Configuring the application involves 3 choices, namely choosing Access Levels, Event Gathering Levels, and if the Application should respond to high-priority request messages.

If we want to allow for high-priority messages to be answered when received independent of the application being busy, we have to run: 

```apl
⎕PROFILE 'start' 'coverage'
```

before starting the application.

The access and event gathering levels we can set while the application is running or when using ``112 ⌶`` to initiate the connection. 

```apl
⍝ AplSource/DemoApplication.aplf
[2] 'POLL:localhost:7000'(112⌶)2 1
```

Take notes of the ``POLL`` at the beginning of the character vector given as the left input to ``112⌶``, this contrasts with the ``SERVE`` command as described in the documentation. This means the application must connect to the client rather than the other way around. 

At current writing, if the application (or rather the interpreter) stands IDLE the communication would stand IDLE as well. The workaround done in the Demo is to have a thread looping with a delay function:

```apl
⍝ AplSource/DemoApplication.aplf
[4] ⎕FX 'LOOP delay;z' 'z←⎕DL delay' '→1'
[5] LOOP&1
```

## Configuring the Client

Configuration of a client depends greatly on what is needed, code standards, and implementation design. In this section the *must have* functionality will be described along with the design choices taken in the specific case of the demo. Therefore will we start this section by giving a short introduction on how to run the Demo. (Feel free to skip over this subsection)

### Running the demo application and client

To start the client simply link and enter the namespace of the demo, and run the function ``Demo`` with out any arguments. 

Open a second interpreter, link and enter namespace again. From here run ``LaunchDemoProcess 0``. The right argument allows for a Ride connection when set to 1. However at  current writing the path the client uses to open Ride is hardcoded and often not able to find the executable to enable this.

Hopefully you can get an idea of what design choices have been made by playing a bit around with this Demo.

### Necessary setup to communicate with protocol

Firstly we need to set up the protocol on the client side. Since HMon is a monitoring protocol, it only provides functionality on the application side, it is our own responsibility to send messages that it can understand, and answer. The demo sets up the IConga protocol:

```apl
⍝ AplSource/Init.aplf
[16] :If 9≠⎕NC 'Conga'
[17]      'Conga'⎕CY'conga'
[18] :EndIf
[19] iConga←Conga.Init'HMONCLIENT'
[20] {}iConga.SetProp'' 'EventMode' 1
[21] magic←iConga.Magic 'HMON'
```

Depending on how you set your client up you would either want to set up a Conga server or connect to all the applications. The Demo is taking the first approach:

```apl
⍝ AplSource/Listen.aplf
[4] d←iConga.Srv '' host port 'BlkText' 32768 ('Magic' magic)
[5] :If 0≢⊃d
[6]     ('Unable to start listener on ',host,':',(⍕port),' - ',3⊃d) ⎕SIGNAL 11
[7] :EndIf
...
[11] monitortid←monitor&1
```

The monitor function is the function which handles all the incoming communication, which has not been explicit called from somewhere else. `monitor` can be seen below.

This concludes the strictly necessary functionality.

```apl
[ 0]monitor listen;z;rc;con;event;data;ns;i;cid;uid;type;tkn;fn;m;removed;done;cb;port;host;json
[ 1] ⍝ While there is anything in the queue, monitor Conga messages
[ 2] ⍝ and release functions waiting on ⎕TPUT
[ 3]
[ 4] listening←listen
[ 5] :While listening∨0≠≢queue
[ 6]     :If 0=⊃z←iConga.Wait'.' 5000
[ 7]         (rc con event data)←4↑z
[ 8]
[ 9]         :Select event
[10]         :Case 'Timeout' ⋄ :Continue
[11]         :Case 'Connect'
[12]             (host port)←(2⊃iConga.GetProp con'PeerAddr')[2 4]
[13]             cid←AddConnection con((-':'⍳⍨⌽host)↓host)port
[14]             Handshake&cid ⍝ Handshake must run on another thread
[15]
[16]         :Case 'Block'
[17]             :If (≢conns)<i←conns[;2]⍳⊂con
[18]                 Log'*** Message received on unexpected connection'
[19]                 Log'===> ',,⍕z
[20]                 :Continue
[21]             :Else
[22]                 cid←conns[i;1]
[23]             :EndIf
[24]
[25]             uid←0
[26]             :Trap 6 11 ⍝ Might be handshake and not JSON data, or JSON data with no UID
[27]                 json←'UTF-8' ⎕UCS ⎕UCS data
[28]                 data←0 ⎕JSON json
[29]                 Log 'GET ',(⍕cid),': ',json
[30]                 tkn←'uid'GetToken uid←2⊃2⊃⎕VFI data[2].UID
[31]             :Else
[32]                 tkn←'cid'GetToken cid
[33]             :EndTrap
[34]
[35]             :Hold 'hmon'
[36]                 cb←0
[37]                 :If (≢queue)<i←queue[;1 2]⍳cid uid
[38]                     Log'*** Unqueued message received'
[39]                     Log'===> ',,⍕z
[40]                     :Continue
[41]                 :ElseIf cb←(0≠≢fn←⊃queue[i;3])∧'Subscribed'≢⊃data ⍝ Callback fn (ignore for "Subscribe" call)
[42]                     (⍎fn)&data            ⍝ Run it in a separate thread
[43]                 :EndIf
[44]
[45]                 :If done←80≠⎕DR data      ⍝ If namespace, check polling state
[46]                     done←data[2].{6::0 ⋄ Interval=0}⍬  ⍝ Polling done
[47]                 :EndIf
[48]                 :If done∨0=≢fn            ⍝   ... or no callback fn
[49]                     queue←(i≠⍳≢queue)⌿queue
[50]                 :EndIf
[51]                 :If ~cb                   ⍝ No callback done, return result via TPUT
[52]                     data ⎕TPUT tkn
[53]                 :EndIf
[54]             :EndHold
[55]
[56]         :CaseList 'Closed' 'Error'
[57]             Log event,': ',con
[58]             :If con≡listener
[59]                 ∘∘∘
[60]             :EndIf
[61]
[62]             :Hold 'hmon'
[63]                  cid←(conns[;1],¯1)[conns[;2]⍳⊂con]
[64]                  removed←(m←cid=queue[;1])⌿queue
[65]                  :If cid≠¯1
[66]                     queue←(~m)⌿queue
[67]                     conns←(m←conns[;1]≠cid)⌿conns
[68]                     hmongrid.(Values CellTypes)←m∘⌿¨hmongrid.(Values CellTypes)
[69]                     Log'===> ',(⍕≢removed),' queue items removed'
[70]                 :EndIf
[71]             :EndHold
[72]             :For uid :In removed[;2]~0
[73]                 'CONNECTION CLOSED'⎕TPUT'uid'GetToken uid
[74]             :EndFor
[75]         :Else
[76]             ∘∘∘
[77]         :EndSelect
[78]
[79]     :Else
[80]         Log'*** Conga.Wait failed: ',⍕z
[81]         ∘∘∘
[82]     :EndIf
[83] :End
```
