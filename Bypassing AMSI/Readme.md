# Bypassing Anti Malware Scan Interface

⇒ &nbsp;&nbsp;When we examined each of the `AMSI Win32 APIs`, we found that they all use the context structure that is created by calling `AmsiInitialize`. However, Microsoft has not documented this context structure.  
⇒ &nbsp;&nbsp;Undocumented functions, structures, and objects are often prone to error, and provide a golden opportunity for security researchers and exploit developers. In this particular case, if we can force some sort of error in this context structure, we may discover a way to crash or bypass AMSI without impacting PowerShell.  
⇒ &nbsp;&nbsp;Since this context structure is undocumented, we will use Frida to locate its address in memory and then use WinDbg to inspect its content. As before, we will open a PowerShell prompt and a trace it with Frida. Then, we'll enter another "test" string to obtain the address of the context structure:  

<img src="https://imgur.com/SGe4pGi.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

⇒ &nbsp;&nbsp;The highlighted section of above reveals the memory address of `amsiContext`. Recall that amsiContext is created when AMSI is initialized so its memory address does not change between scans, allowing us to inspect it easily with WinDbg.  
⇒ &nbsp;&nbsp;As a next step, we'll open WinDbg, attach to the PowerShell process, and dump the memory contents of the context structure as shown  

<img src="https://imgur.com/16MFmQa.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

⇒ &nbsp;&nbsp;We don't know the size of the context structure but we find that the first four bytes equate to the `ASCII` representation of "AMSI". This seems rather interesting and might be usable since this string is likely static between processes.  
⇒ &nbsp;&nbsp;If we can observe the context structure in action in the AMSI APIs, we may be able to determine if the first four bytes are being referenced in any way. To do this, we'll use the unassemble command in WinDbg along with the AmsiOpenSession function from the AMSI module:  

<img src="https://imgur.com/HdVAZ1U.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

⇒ &nbsp;&nbsp;The fourth line of assembly code is interesting as it compares the contents of a memory location to the four static bytes we just found inside the context structure.  
⇒ &nbsp;&nbsp;According to the 64-bit calling convention, we know that `RCX` will contain the function's first argument. The first argument of `AmsiOpenSession` is exactly the context structure according to its function prototype, which means that a comparison is performed to check the header of the buffer.  
⇒ &nbsp;&nbsp;Although we don't know much about this context structure, we observe that the first four bytes equate to the ASCII representation of "AMSI". After the comparison instruction, we find a `conditional jump` instruction, JNE, which means "jump if not equal".  
⇒ &nbsp;&nbsp;If the header bytes are not equal to this static DWORD, the conditional jump is triggered and execution goes to offset 0x4B inside the function. Let's use WinDbg to display the instructions at that address:  

<img src="https://imgur.com/qCaQ150.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

⇒ &nbsp;&nbsp;The conditional jump leads directly to an exit of the function where the static value `0x80070057` is placed in the EAX register. On both the 32-bit and 64-bit architectures, the function return values are returned through the `EAX/RAX` register.  
⇒ &nbsp;&nbsp;If we revisit the function prototype of AmsiOpenSession as given, we notice that the return value type is `HRESULT`.  

<img src="https://imgur.com/H0d2346.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

⇒ &nbsp;&nbsp;`HRESULT` values are documented and can be referenced on MSDN where we find that the numerical value 0x80070057 corresponds to the message text `E_INVALIDARG`. The message text, while not especially verbose, indicates that an argument, which we would assume to be amsiContext, is invalid.  
⇒ &nbsp;&nbsp;In short, this error occurs if the context structure has been corrupted. If the first four bytes of amsiContext do not match the header values, AmsiOpenSession will return an error. What we don't know is what effect that error will cause. In a situation like this, there are typically two ways forward.  
⇒ &nbsp;&nbsp;The first is to trace the call to AmsiOpenSession that returns this error and try to figure out where that leads. This could become very time-consuming and complex. The second, and much simpler, approach is to force a failed `AmsiOpenSession` call, let the execution continue, and observe what happens in our Frida trace. Let's try this approach first.  
⇒ &nbsp;&nbsp;In order to force an error, we'll place a breakpoint on AmsiOpenSession and trigger it by entering a PowerShell command. Once the breakpoint has been triggered, we'll use ed to modify the first four bytes of the context structure, and let execution continue:  

<img src="https://imgur.com/NTZPNVz.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

⇒&nbsp;&nbsp; After overwriting the AMSI header value, we'll continue execution, which generates exceptions:  

<img src="https://imgur.com/xYpuJiQ.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

⇒ &nbsp;&nbsp;According to this output, `AmsiOpenSession()` has exited. This could indicate that AMSI has been shut down.  
⇒ &nbsp;&nbsp;To test this, we'll enter the `'amsiutils'` string that was previously flagged as malicious:  

<img src="https://imgur.com/ihGKUNK.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

⇒ &nbsp;&nbsp;This time, none of the hooked AMSI APIs are called and our command is not flagged. By `corrupting the amsiContext header`, we have effectively shut down AMSI without affecting PowerShell. We have effectively bypassed AMSI. Very nice.  
⇒ &nbsp;&nbsp;Although this method is effective, it relies on manual intervention with WinDbg. Let's try to implement this bypass directly from PowerShell with reflection.  
⇒ &nbsp;&nbsp;PowerShell stores information about AMSI in managed code inside the `System.Management.Automation.AmsiUtils` class, which we can enumerate and interact with through reflection.    
⇒ &nbsp;&nbsp;As previously discussed, a key element of reflection is the GetType method, which we'll invoke through `System.Management.Automation.PSReference`,ref also called [Ref].  
⇒ &nbsp;&nbsp;GetType accepts the name of the assembly to resolve, which in this case is System.Management.Automation.AmsiUtils. Before we execute any code, we'll close the current PowerShell session and open a new one to re-enable AMSI.  
⇒ &nbsp;&nbsp;Note that using a large number of AMSI trigger strings while testing may cause a "panic" in Windows Defender and it will suddenly consider everything malicious. At this point, the only remedy is to reboot the system.  

<img src="https://imgur.com/vREgggm.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

⇒ &nbsp;&nbsp;Sadly, Windows Defender and AMSI are blocking us from obtaining a reference to the class due to the malicious `'AmsiUtils'` string. Instead, we can locate the class dynamically.  
⇒ &nbsp;&nbsp;We could again attempt to bypass Windows Defender with a split string like `'ams'+'iUtils'` (as some public bypasses do), but Microsoft regularly updates the signatures and this simple bypass may eventually fail.  
⇒ &nbsp;&nbsp;Instead, we'll attempt another approach and `loop the GetTypes method`, searching for all types containing the string `"iUtils"` in its name:  

<img src="https://imgur.com/8i9jtmq.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

⇒&nbsp;&nbsp; Armed with a handle to the AmsiUtils class, we can now invoke the GetField method to enumerate all objects and variables contained in the class. Since `GetFields` accepts filtering modifiers, we'll apply the NonPublic and Static filters to help narrow the results:

<img src="https://imgur.com/na6bWC3.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

⇒&nbsp;&nbsp; As we will soon realize, the amsiContext field contains the unmanaged amsiContext buffer. However, we can not reference the field directly since amsiContext also contains a malicious `"amsi"` string. We'll again loop through all the fields, searching for a name containing `"Context"`:

<img src="https://imgur.com/s2JOxHI.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

⇒ &nbsp;&nbsp;Although we managed to obtain the amsiContext field without triggering AMSI, the output contains a very large integer. Converting this to hexadecimal produces `0x1609A791020`, which looks like a valid memory address.  
⇒ &nbsp;&nbsp;To verify our theory that this is indeed the address of the amsiContext buffer, we'll open and attach WinDbg and dump the memory at address `0x1609A791020`:  



<img src="https://imgur.com/WCc7A8m.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

⇒ &nbsp;&nbsp;The first four bytes at the dumped address contain the AMSI header values, indicating that this is very likely the `amsiContext buffer`  .
⇒ &nbsp;&nbsp;Now, let's put this all together. We'll recreate each of the steps and use Copy to overwrite the amsiContext header by copying data (four zeros) from managed to unmanaged memory:  

<img src="https://imgur.com/2GIcvJu.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

<img src="https://imgur.com/2GIcvJu.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

⇒ &nbsp;&nbsp;We do not get any output from the executed commands, but we can inspect our work by switching to WinDbg, forcing a break through ` Debug > Break` and dumping the content of the `amsiContext buffer`:  

<img src="https://imgur.com/7U2IUTw.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

⇒ &nbsp;&nbsp;This output indicates that the context structure header was indeed overwritten, which should force `AmsiOpenSession` to error out.   
⇒ &nbsp;&nbsp;Next, let's continue execution in the debugger, switch back to PowerShell, and enter the malicious 'amsiutils' string:  

<img src="https://imgur.com/gNYCKkh.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

<img src="https://imgur.com/VB6zPLR.png" height="70%" width="75%" alt="Bypassing Amsi Steps"/>

⇒ &nbsp;&nbsp;The string was not flagged. Excellent. We can combine this bypass into a `PowerShell one-liner`  
⇒ &nbsp;&nbsp;Not only is this bypass working, but it is difficult to blacklist now that we've removed explicit signature strings and `dynamically resolved` the types and fields.  
⇒ &nbsp;&nbsp;This is working well, but it's not the only approach. We'll work through another bypass in the next section.  


















