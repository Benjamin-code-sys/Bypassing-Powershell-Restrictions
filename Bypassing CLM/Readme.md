# Bypassing Powershell Constrained Language Mode (csharp)

⇒ &nbsp;&nbsp;For this we will leverage `InstallUtil`, a command-line utility that allows us to `install` and `uninstall` server resources by executing the installer components in a specified assembly. This Microsoft-supplied tool obviously has legitimate uses, but we can abuse it to execute `arbitrary C# code`. Our goal is to reintroduce our PowerShell shellcode runner tradecraft in an AppLocker-protected environment.  
⇒ &nbsp;&nbsp;To use `InstallUtil` in this way, we must put the code we want to execute inside either the install or uninstall methods of the installer class.  
⇒ &nbsp;&nbsp;We are only going to use the `uninstall method` since the install method requires `administrative privileges` to execute.  
⇒ &nbsp;&nbsp;Using the MSDN documentation as a guide, we can build the following proof-of-concept:  

<img src="https://imgur.com/yUsMBOg.png" height="70%" width="75%" alt="Bypassing CLM Steps"/>

⇒ &nbsp;&nbsp;There are a few things to note about this code. First, the `System.Configuration.Install` namespace is missing an assembly reference in Visual Studio. We can add this by again right-clicking on References in the Solution Explorer and choosing Add References.... From here, we'll navigate to the Assemblies menu on the left-hand side and scroll down to `System.Configuration.Install`, as shown  

<img src="https://imgur.com/FelwsUu.png" height="70%" width="75%" alt="Bypassing CLM Steps"/>

⇒ &nbsp;&nbsp; Once the assembly reference has been added the displayed errors are resolved. Although our code uses both the Main method and the Uninstall method, content in the Main method is not important in this example. However, the method itself must be present in the executable.  
⇒ &nbsp;&nbsp; Since the content of the Main method is not part of the application whitelisting bypass, we could use it for other purposes, like bypassing antivirus.  
⇒ &nbsp;&nbsp; Inside the Uninstall method, we can execute `arbitrary C# code`. In this case, we will use the custom runspace code we developed in the previous section. The combined code is shown  

<img src="https://imgur.com/BCVeoIv.png" height="70%" width="75%" alt="Bypassing CLM Steps"/>

⇒&nbsp;&nbsp; With the executable compiled and copied to the Windows 10 victim machine, we'll execute it from an administrative command prompt:  

<img src="https://imgur.com/QOsLWFu.png" height="70%" width="75%" alt="Bypassing CLM Steps"/>

⇒&nbsp;&nbsp; As shown in the output, the Main method executed. If we run it from a non-administrative command prompt, AppLocker blocks it.  
⇒&nbsp;&nbsp; To trigger our constrained language mode bypass code, we must invoke it through `InstallUtil` with `/logfile` to avoid logging to a file, `/LogToConsole=false` to suppress output on the console and `/U` to trigger the Uninstall method:  

<img src="https://imgur.com/VPUDt1S.png" height="70%" width="75%" alt="Bypassing CLM Steps"/>

⇒&nbsp;&nbsp;  The output shows that InstallUtil is allowed to execute. It started the `.NET Framework Installation utility` and the test.txt output shows that our PowerShell script executed without restrictions. Excellent!  
⇒&nbsp;&nbsp;  At this point, it would be possible to reuse this tradecraft with the Microsoft Word macros we developed in a previous module since they are not limited by AppLocker. Instead of using WMI to directly start a PowerShell process and download the shellcode runner from our Apache web server, we could make `WMI execute InstallUtil` and obtain the same result despite `AppLocker`.  
⇒&nbsp;&nbsp;  There is, however, a slight issue; the `compiled C#` file has to be on disk when InstallUtil is invoked. This requires two distinct actions. First, we must download an executable, and secondly, we must ensure that it is not flagged by antivirus, neither during the download process nor when it is saved to disk. We could use `VBA code` to do this, but it is simpler to rely on other `native Windows binaries`, which are `whitelisted` by default.  
⇒&nbsp;&nbsp;  To attempt to bypass anitvirus, we are going to obfuscate the executable while it is being downloaded with Base64 encoding and then decode it on disk. Well use the native certutil tool to perform the encoding and decoding and bitsadmin for the downloading. By using native tools in unexpected and interesting ways, we will be `"Living Off The Land"`.    

<img src="https://imgur.com/kYGLGTR.png" height="70%" width="75%" alt="Bypassing CLM Steps"/>

⇒&nbsp;&nbsp; Now that the binary has been `Base64-encoded`, we'll copy it to the web root of our Kali machine and ensure that Apache is running. Then we'll use bitsadmin to download the encoded file.  
⇒&nbsp;&nbsp; `Certutil` can also be used to download files over `HTTP(S)`, but this triggers antivirus due to its widespread malicious usage.  
⇒&nbsp;&nbsp; To download the file, we'll specify the `/Transfer` option along with a custom name for the transfer and the download URL:  

<img src="https://imgur.com/sQZ0j3N.png" height="70%" width="75%" alt="Bypassing CLM Steps"/>

⇒ &nbsp;&nbsp;With the file downloaded we can decode it with `certutil -decode`:    

<img src="https://imgur.com/I1YDUBd.png" height="70%" width="75%" alt="Bypassing CLM Steps"/>

⇒&nbsp;&nbsp; As shown, we executed the decoded executable with `InstallUtil` and bypassed both AppLocker's executable rules and PowerShell's `constrained language mode`.  
⇒&nbsp;&nbsp; Since all of these commands are executed sequentially we can combine them on the command line through the && syntax:  

<img src="https://imgur.com/yvksBfT.png" height="70%" width="75%" alt="Bypassing CLM Steps"/>

⇒&nbsp;&nbsp; Our bypasses were again successful. Very Nice.  




