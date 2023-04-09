---
title: "Collecting the DLLs Required by an MSYS2 Binary"
date: 2022-04-21T09:27:15Z
---

After many unsuccessful attempts of writing a third blog post, I just wanted to
use this opportunity to share something useful I found after dabbling with
porting an _SDL2_ app to _Windows_ via _MSYS2 MinGW x64_.

TL;DR: You can collect all the _MinGW_ DLLs your EXE file needs like this:

    ldd my-cool-program.exe | grep /mingw64 | awk '{print $3}' | xargs -i cp {} .

You see, in the magical Linux realm you'll never really worry about shipping
programs in binary form. A nice tarball with a well documented build system is
often regarded as good enough. If your program is actually used by people, its
likely that distributions pick it up into their repositories. Or you use stuff
like _Flatpak_ or _AppImage_ if you want to provide binaries yourself. But
over in Windows-land providing .exe files is a normal part of distributing your
programs.

Now, if you use any special libraries they are often dynamically linked and
their code is stored externally to your .exe as a .dll. Some .dll files are
simply stored somewhere in System32 and are thus available everywhere, but
others have to be shipped manually alongside your program.

---

Now let me go over a short tangent praising the efforts the developers of
_MinGW64_ have made. I wrote a little game in C using Lua and SDL2. On Linux,
building that game was simple. I'd use GNU/Make, _pkg-config_ and my distro's
package manager to collect the dependencies and tie everything together. I'd
dreaded the Windows port but eventually gave it a shot. And holy shit. _MSYS2_
provides EXACTLY THE SAME workflow!

The packages' names are a bit more arcane sounding (e.g.:
`mingw-w64-x86_64-gcc`, `mingw-w64-x86_64-SDL2`, `mingw-w64-x86_64-lua`),
because MSYS2 provides multiple toolchains and thus the packages have to be
more explicit in their naming. But after installing everything and running
`make` I had a working .exe file! Well... It was working after I fixed a
segfault that I didn't catch under Linux anyways. But that one's on me.

Running the .exe in MSYS2's shell worked fine, but starting it outside of the
shell resulted in most DLLs being unavailable. That's because, all non-system
DLLs are not provided to the program by default. That includes stuff like Lua's
or SDL2's DLL files. They are stored in the path `/mingw64/bin/`. Now, you
COULD go ahead and manually copy them, but why do that when that task can be
automated?

---

MinGW64 provides a version of the `ldd` program, which spits out all the
dynamically linked libraries required by a program. Its output may look
something like this:

    $ ldd my-cool-game.exe
            ntdll.dll => /c/Windows/SYSTEM32/ntdll.dll (0x7ffa09e30000)
            KERNEL32.DLL => /c/Windows/System32/KERNEL32.DLL (0x7ffa09d30000)
            KERNELBASE.dll => /c/Windows/System32/KERNELBASE.dll (0x7ffa075f0000)
            msvcrt.dll => /c/Windows/System32/msvcrt.dll (0x7ffa08c00000)
            SHELL32.dll => /c/Windows/System32/SHELL32.dll (0x7ffa08cb0000)
            msvcp_win.dll => /c/Windows/System32/msvcp_win.dll (0x7ffa07550000)
            ucrtbase.dll => /c/Windows/System32/ucrtbase.dll (0x7ffa07d90000)
            USER32.dll => /c/Windows/System32/USER32.dll (0x7ffa09690000)
            win32u.dll => /c/Windows/System32/win32u.dll (0x7ffa07d60000)
            GDI32.dll => /c/Windows/System32/GDI32.dll (0x7ffa08a50000)
            gdi32full.dll => /c/Windows/System32/gdi32full.dll (0x7ffa078c0000)
            SDL2_mixer.dll => /mingw64/bin/SDL2_mixer.dll (0x7ff9f7fd0000)
            lua54.dll => /mingw64/bin/lua54.dll (0x7ff9f0e80000)
            libwinpthread-1.dll => /mingw64/bin/libwinpthread-1.dll (0x7ff9f7210000)
            SDL2.dll => /mingw64/bin/SDL2.dll (0x7ff9e39f0000)
            (...)

Most referenced DLLs reside in Window's System32 folder. They are provided by
the system and will always be there. So you don't have to worry about providing
them. The DLLs in `/mingw64/bin/...` however need to be placed alongside your
.exe file if people should be able to run your game via double-clicking it.

Cool, so that's where our automation can begin! First step is super simple:
let's filter out the non-system DLLs using `grep`:

    ldd my-cool-program.exe | grep /mingw64

Might not be _super_ robust, but I _highly_ doubt that the string "/mingw64"
will ever show up in a Windows system DLL's path.

Next, we only care about the DLL's path. So we use `awk` to cut out that
portion of the lines. If we regard the spaces as delimiting characters of text
columns, the full path is in the third column (The first is just the filename,
and the second is "=>"). The `awk` command `awk {print $3}` gives us this third
column, so we can just append it to our shell command:

    ldd my-cool-program.exe | grep /mingw64 | awk '{print $3}'

By now, our shell command spits our a list of full paths of all non-system DLLs
dynamically linked to our program. Cool! But we want to automate the whole
thing, so lets add a final call to `xargs`, to copy all files in this list into
our current directory. Here, I'll use `xargs -i cp {} .`. The `-i cp {} .`
parameter means that, for every file in the list, we call `cp`, pass the DLL's
path as the first parameter and the target directory `.` as the second
parameter.

Here is the final call:

    ldd my-cool-program.exe | grep /mingw64 | awk '{print $3}' | xargs -i cp {} .

---

Cool! Now go put this in your CI script and automate packaging your Windows
releases. Or, if you're like me, make exactly one release and wonder why you
spent so much time on figuring this out when you could have just collected the
files manually once and then forget about it ARGH
