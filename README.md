# faxhell ("Fax Shell")
A Bind Shell Using the Fax Service and a DLL Hijack
Writeup: https://windows-internals.com/faxing-your-way-to-system/

## How to use
* Build ualapi.dll and place in c:\Windows\System32
* Start the Fax service, which will load the dll and call the export UalStart. UalStart will create a Thread Pool worker thread that will open a handle to RpcSs, find a SYSTEM token and impersonate it, then open a socket and wait for connection.
* Connect to the socket on port 9299 (ncat.exe <ip> 9299) and insert "password": "let me in\n"
* Worker thread will open a cmd.exe process under the DcomLaunch service with SYSTEM privileges.
* Win!
  
## EDR / AV evasion
* Using a service that is not commonly known and not monitored or flagged as suspicious by EDR vendors.
* Using Windows Thread Pool to do setup, making stacks harder to read and avoid easy "hints" that something suspicious is happening.
* Not elevating the main thread to SYSTEM and reverting back to NETWORK SERVICE very quickly, only doing one API call as SYSTEM, makes the process look a lot less suspicious.
* Using uncommon socket APIs that don't make stacks look suspicious and avoid EDR detections.
* Creating shell under DcomLaunch service (which is already a SYSTEM service) and not under Fax service makes it look a lot more natural and avoids a very suspicious-looking process tree.
* Windows bug makes it look as if our socket belongs to the Fax service, and not to DcomLaunch or cmd.exe. If we kill the Fax service it looks like socket belongs to System.
