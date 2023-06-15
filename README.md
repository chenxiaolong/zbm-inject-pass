# zbm-inject-pass

zbm-inject-pass is a custom build of [ZFSBootMenu](https://zfsbootmenu.org) that avoids the need for a second password prompt from the OS when booting from an encrypted ZFS dataset.

It does so by saving the passphrase from the bootloader's interactive prompt to a key file and then injecting the key file into the OS's initramfs prior to kexec. This is done in memory and the passphrase never touches the disk.

## Installation

1. Make sure either podman or docker is installed.

2. Clone both the ZFSBootMenu repository and this repository.

    ```bash
    git clone https://github.com/zbm-dev/zfsbootmenu.git
    git clone https://github.com/chenxiaolong/zbm-inject-pass.git
    ```

    By default, this will build against the latest commit in zfsbootmenu. To build against the latest stable version instead, run:

    ```bash
    tag=$(curl -sS https://api.github.com/repos/zbm-dev/zfsbootmenu/releases/latest | jq -r .tag_name)
    git -C zfsbootmenu checkout "${tag}"
    ```

3. Create a custom ZFSBootMenu image.

    ```bash
    ./zfsbootmenu/zbm-builder.sh -l zfsbootmenu -b zbm-inject-pass
    ```

4. The newly built images will be written to `zbm-inject-pass/build/`.

## Usage

By default, zbm-inject-pass' functionality is disabled. The bootloader should behave exactly like regular ZFSBootMenu. To enable zbm-inject-pass:

1. Set the dataset's `keylocation` to a file path. The file does not need to actually exist.

    ```bash
    zfs set keylocation=file:///some/arbitrary/path <dataset>
    ```

2. Enable zbm-inject-pass on the dataset.

    ```bash
    zfs set zbm-inject-pass:enable=on <dataset>
    ```

That's it! On boot, the passphrase entered into ZFSBootMenu's interactive prompt will be injected into the OS's initramfs image at the `keylocation`. When the OS mounts the dataset itself, it'll be able to find the injected key file.

## How it works

To reduce the chance of things breaking due to future changes, zbm-inject-pass doesn't hook into ZFSBootMenu at all. Instead, it renames the `/usr/bin/zfs` and `/usr/bin/kexec` binaries during the build process and adds wrappers in their place.

When `zfs load-key -L prompt <encryption root>` is run, if the dataset has `zbm-inject-pass:enable` set to `on`, then the wrapper will save the passphrase to a key file in `/tmp/zbm-inject-pass/<keylocation>`. Similarly, when `zfs unload-key` is run, the corresponding key file is removed.

When `kexec --initrd <initramfs> <...>` is run, if `/tmp/zbm-inject-pass` has any key files, then the wrapper will store the keys in a new cpio archive and append it to the end of the OS's original initramfs image before passing it to the actual kexec command. This generated initramfs image is stored only in memory and is never written to disk.

While there's not much that can go wrong with these wrappers, the build process is a bit fragile and depends on implementation details of dracut/mkinitcpio. Neither tool supports renaming files (eg. `/usr/bin/zfs` -> `/usr/bin/zfs.orig`), so this project resorts to doing some ugly hacks to make it work. These might need to be updated for future dracut/mkinitcpio changes.

## License

This project is licensed under the same MIT license as ZFSBootMenu. Please see [`LICENSE`](./LICENSE) for details.
