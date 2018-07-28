To ensure build/install time isolation between the host system and the target system, it's recommended to use `chroot (8)` environment.
If such isolation is not provided, several threats can entail:

- Host system may get corrupted by accidentally installing target tools or libraries into its system directories.
- Target system toolchain can accidentally get dependent on foreign tools or libraries, which will prevent it from achieving self-hostedness.
