# faxhell ("Fax Shell")
A Proof-of-Concept bind shell using the `Fax` service and a DLL hijack based on `Ualapi.dll`.

See our writeup at: https://windows-internals.com/faxing-your-way-to-system/

![Obligatory Demo](https://windows-internals.com/wp-content/uploads/2020/04/port_bind_connect.png)

## How to use
* Build `Ualapi.dll` and place in `c:\windows\system32`
* Start the `Fax` service, which will load the DLL and call the export `UalStart`. `UalStart` will queue a thread pool work item that will open a handle to `RpcSs`, find a `SYSTEM` token, and then impersonate it. Afterward, it will create a socket on the local endpoint address, bind it to port `9299`, and then asynchronously wait for a connection using a thread pool I/O completion port.
* Connect to the socket on port 9299 using your favorite client (such `nc(at).exe <ip> 9299`) and then type `let me in` and press `ENTER`. If you're writing custom code, make sure to send the string `let me in\n`.
* The I/O completion packet will then wake up the thread pool callback, which will start a `Cmd.exe` process under the `DcomLaunch` service with `SYSTEM` privileges, binding its input and output handles to the newly created socket.
* Win!
  
## EDR / AV evasion
* Uses a service that is not commonly known and not monitored or flagged as suspicious by EDR vendors.
* Uses the Windows thread pool API to do setup, making stacks harder to read, offloading work through multiple threads, and avoiding easy "hints" that something suspicious is happening.
* The lifetime of the impersonated tokens is very small, and only the worker thread ever runs as `SYSTEM`, reverting back to `NETWORK SERVICE` very quickly and after only doing one API call. This helps reduce the chance of getting caught by various scanners.
* Uses uncommon socket APIs that make the import table less suspicious and avoids EDR detections, IOCTL hooks, and LSPs.
* Creates the bind shell under the `DcomLaunch` service (which is already a `SYSTEM` service) and not under the `Fax` service, making it look a lot more natural and avoiding a very suspicious-looking process tree.
* Leverages a Windows bug that makes it look as if our socket belongs to the `Fax` service, and not to `DcomLaunch` or `Cmd.exe`. If we kill the `Fax` service it looks like socket belongs to `System`.

## Caveats
This isn't meant to be a drop-in, undetectable, malicious, weaponized shell:
* It is only a bind shell, which most firewalls will prevent. Opening firewall rules, or using a reverse bind shell, or doing communications over a common port such as `80` or `443` would work better.
* Other services, notably the `Spooler`, also load `Ualapi.dll`. While the system behaves fine if the `Fax` service is "stuck" in the `SERVICE_START_PENDING` state, this will cause issues in `Spoolsv.exe`.
* There's probably bugs/memory leaks in the PoC -- we tried our best to make things production quality, but we did not run things through Application Verifier or asan.
