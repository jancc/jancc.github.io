---
title: "Ephemeral Downloads"
date: 2019-09-19T09:27:04Z
---

This is just a quick tipp. I added the following line to my `/etc/fstab` file:

    tmpfs /home/jan/Downloads tmpfs rw,nodev,noexec,size=1G 0 0

It mounts my downloads directory onto an in-memory filesystem. This effectively
makes my downloads only stay in RAM. I had the problem in the past that this
directory would balloon in size because I'd never clean it. Now it will always
be cleaned on reboot because RAM can't hold data without being powered.

It has some other useful side effects too. As you may noticed there are some
other flags added to the mount. `nodev` is obvious. `noexec` makes it
impossible to execute anything in this folder. When I'm downloading binaries
in order to execute them I want to force myself into moving them somewhere else
first. Also while writing this... `nosuid` is kinda redundant now, isn't it? Oh
well.

The size is limited to a single gigabyte. Larger files should likely be
archived somewhere else directly, because I don't want to download those
multiple times. Is is no problem, by the way, to overprovision the mount's
size. It does not reserve the full size on memory, but it grows dynamically.
Therefore you could extend this idea onto many more mountpoints.

Theoretically this is also a great solution should your net-connection be
faster than your hard drive's write speed. But I live in Germany so this isn't
something I'd need to ever worry about lol.

