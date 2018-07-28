To ensure build/install time isolation between the host system and the target system, it's recommended to use `chroot (8)` environment.
If such isolation is not provided, several threats can entail:

- Host system may get corrupted by accidentally installing target tools or libraries into its system directories.
- Target system toolchain can accidentally get dependent on foreign tools or libraries, which will prevent it from achieving self-hostedness.

You need some shell in the new root environment to be able to execute interactive commands. It can be any shell that you specify on the command line to `chroot (8)` or `$SHELL` if nothing is specified. In my case it is `/bin/bash`, so further conversation will use `bash` as an example, but all applies to other shells as well.

Unfortunately, simply copying `bash` from host to target doesn't work.
`bash` depends on several dynamic libraries, which needs to be present in the new root environment.
Dependencies can be figured out by running `ldd /bin/bash` command.
Below is how the dependency tree looks like on Debian 9 "sid".

```
/bin/bash
|_ /lib/x86_64-linux-gnu/libtinfo.so.5 (shared low-level terminfo library for terminal handling)
   |_ /lib/x86_64-linux-gnu/libc.so.6
|_ /lib/x86_64-linux-gnu/libdl.so.2    (dynamic linking library)
   |_ /lib/x86_64-linux-gnu/libc.so.6
|_ /lib/x86_64-linux-gnu/libc.so.6
```

Besides that, every executable and library also depends on `linux-vdso.so.1` `vdso (7)` and dynamic linker `/lib64/ld-linux-x86-64.so.2`.

All those libraries should be copied to new root environment together with the shell binary in order for `chroot (8)` to succeed.

NOTE: for `zsh` shell, you'll need all the same dependencies, but also `/lib/x86_64-linux-gnu/libcap.so.2` (user-space interfaces to the POSIX 1003.1e capabilities implemented in Linux) and `/lib/x86_64-linux-gnu/libm.so.6` (C math library).
