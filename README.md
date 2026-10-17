# InnoMedia Password Generator

A small, self-contained browser utility that generates the device-specific `innomediaadmin` service password used by at least some InnoMedia VoIP devices.

**Tested with:** InnoMedia MTA8328-1N, hardware A4.1, firmware 1.0.10.3.

Other InnoMedia models and firmware versions may use the same algorithm, but have not been verified.

## Online Password Generator

[Open the InnoMedia Password Generator](https://wormfood.github.io/InnoMedia_Password_Generator/innomedia_password_generator.html)

## Why this exists

I was given an InnoMedia MTA8328-1N that I wanted to reuse as a normal VoIP adapter. The unit worked, but at first I had no useful way to administer it.

One part of the problem eventually turned out to be almost embarrassingly simple: the normal administrator account is `Admin`, with a capital A. I had been trying `admin`.

By the time I discovered that, I had already gone considerably farther.

I removed the SPI flash ROM and dumped it so I could inspect the firmware directly. I connected to the serial console and tracked down the password for the serial bootloader. From there I temporarily bypassed the normal startup by adding:

```text
init=/bin/sh
```

to the Linux boot arguments.

That gave me an early root shell. Since the normal initialization scripts had not run, I mounted the required filesystems manually and started examining the persistent configuration, account data, and management software.

That led me to the files used for the normal accounts and, eventually, to the code that handles the hidden `innomediaadmin` service account.

## Serial console and root shell

On the tested MTA8328-1N, the four-pin serial header is a 3.3 V TTL UART.

The pin **closest to the edge of the PCB is +3.3 V**. From the PCB edge moving inward, the pinout is:

```text
+3.3V
TX
RX
GND
```

Equivalently, moving toward the PCB edge:

```text
GND
RX
TX
+3.3V
```

For normal serial access, connect ground and cross TX/RX between your USB serial adapter and the MTA. The +3.3 V pin is not needed for normal UART communication.

Serial settings:

```text
115200 baud
8 data bits
No parity
1 stop bit
```

or simply:

```text
115200 8N1
```

### Bootloader access

Interrupting the bootloader presents a password prompt.

On the tested firmware:

```text
Bootloader password: !nn0M3d1@
```

There is only **one password attempt**. If it is entered incorrectly, reboot the unit and try again.

There is another important quirk: this bootloader can miss characters if they are sent too quickly. Long commands pasted at normal terminal speed may be silently mangled. Use terminal paste throttling, add a delay between characters, or enter long commands slowly. This matters especially when changing the kernel command line.

Do not use `saveenv` unless you intentionally want to make a bootloader change persistent.

### Getting the early root shell

The following is the actual procedure used on the tested MTA8328-1N.

This unit had firmware **partition 2 active**, with `rootfs_2` at `/dev/mtdblock3`.

At the `im-boot #` prompt I entered:

```text
fsload 3:
setenv bootargs console=ttyS0,115200n8 root=/dev/mtdblock3 model=MTA8328-1N mtdparts=mtdparts=spi_flash:256k@0(im-boot),16k@240k(bootenv),7552k@256k(rootfs_1),7552k@7808k(rootfs_2)ro,-(sysconf) ubver=IM8328W_01.02 init=/bin/sh
bootm
```

`fsload 3:` loads the kernel from the active `rootfs_2` partition on this unit. The `setenv` command reproduces the normal kernel command line but adds:

```text
init=/bin/sh
```

which tells Linux to start `/bin/sh` instead of the normal init process.

**Do not run `saveenv`.** The change is intentionally temporary and disappears after a reboot.

Also remember the bootloader input problem mentioned above. The long `setenv` command is exactly the kind of command that can be corrupted when pasted too quickly. Use terminal paste throttling or enter it slowly, and inspect the command on the screen before entering `bootm`.

After `bootm`, Linux should eventually stop at something similar to:

```text
BusyBox v1.13.4 (...) built-in shell (ash)
Enter 'help' for a list of built-in commands.

bin/sh: can't access tty; job control turned off
#
```

The `can't access tty` message is expected in this early shell.

Because the normal startup scripts were bypassed, `/proc` and the persistent configuration filesystem were not mounted. I used:

```sh
mount -t proc proc /proc
cat /proc/mtd
```

On the tested unit, `/proc/mtd` showed:

```text
mtd0: 00040000 00001000 "im-boot"
mtd1: 00004000 00001000 "bootenv"
mtd2: 00760000 00001000 "rootfs_1"
mtd3: 00760000 00001000 "rootfs_2"
mtd4: 00100000 00001000 "sysconf"
mtd5: 01000000 00001000 "spi_flash"
```

I then mounted the persistent JFFS2 configuration partition:

```sh
mount -t jffs2 /dev/mtdblock4 /sysconf
```

The files under `/sysconf` then became available for examination.

These partition numbers are documented exactly as used on the tested unit. If another device has a different active firmware partition or flash layout, check its bootloader environment and `/proc/mtd` rather than blindly assuming the same numbers.

### An easier shell path

After all of that, I found that the running management CLI has a hidden command:

```text
console on
```

It prompts for a shell password. On the tested firmware:

```text
Shell password: !nn0M3d1@
```

That launches `/bin/sh` directly from the running system, so once you have CLI access there is normally no reason to use the `init=/bin/sh` boot method again.

## Known credentials on the tested firmware

These were recovered or confirmed on the tested MTA8328-1N, hardware A4.1, firmware 1.0.10.3:

| Access | Username / command | Password | Notes |
| --- | --- | --- | --- |
| Normal administrator | `Admin` | `password` | Case-sensitive username. Used for normal administration. |
| Normal user | `user` | `welcome` | Limited account. |
| Service CLI | `innomediaadmin` | Device-specific, generated from the MAC address | Works at the management CLI over the serial console, Telnet, and SSH. Does **not** work at the web login. |
| Hidden shell from management CLI | `console on` | `!nn0M3d1@` | Launches `/bin/sh` on the tested firmware. |
| Factory menu | `f` / Factory | `inno1901fact` | Opens the manufacturing/service Factory menu. |
| Serial bootloader | bootloader password prompt | `!nn0M3d1@` | IM-Boot allows only one password attempt per boot. |

The service-account password is the only device-specific credential in this table. This generator calculates that password from the MAC address.

## The `innomediaadmin` account

The password for `innomediaadmin` is generated from the device's MAC address. It is not simply a universal password.

On the tested MTA8328-1N, the `innomediaadmin` login can be used with the management CLI through:

- the serial console
- Telnet
- SSH

It does **not** work with the web interface login.

The firmware also has a special `showhint` path for this account. That code generates the same device-specific password and invokes a QR-code helper using an InnoMedia URL. That strongly suggests that `innomediaadmin` was intended as a service or support mechanism.

This utility reproduces the password calculation locally. It does not use InnoMedia's QR-code service and does not send the MAC address or password anywhere.

I figured this could be a handy little tool for someone who owns one of these devices and wants to use the service login, particularly if the normal administrative password has been forgotten.

## Privacy and offline use

The generator is entirely client-side JavaScript.

There are:

- no server-side components
- no external JavaScript libraries
- no external fonts
- no analytics
- no network requests made by the generator after the page is loaded

Your generated password is never stored. The last MAC address you enter is saved only in the browser's `localStorage` so it can be restored the next time you open the generator.

No cookie is used. Unlike cookies, `localStorage` is not automatically sent to the web server with HTTP requests.

When using the GitHub Pages version, the saved MAC remains in that browser's local storage for the `wormfood.github.io` origin. If you save the HTML file and open it directly with `file://`, browsers generally keep local storage on the local machine, but persistence and scoping for `file://` URLs are browser-dependent.

`innomedia_password_generator.html` is completely self-contained. You can save the page and open it locally, and the generator will continue to work without an Internet connection.

## Using it

Open `innomedia_password_generator.html` in a browser.

Enter the device MAC address in the six hexadecimal fields. The password is generated immediately as you type.

You can also paste a complete MAC address in any common format:

```text
00:10:99:12:34:56
00-10-99-12-34-56
001099123456
```

The page will split it into the six fields automatically.

## Compatibility

This has been verified on:

```text
Model:     InnoMedia MTA8328-1N
Hardware:  A4.1
Firmware:  1.0.10.3
```

It may work with other InnoMedia devices.

I suspect the firmware version itself will not matter as long as InnoMedia retained the same service-account password routine, but that has not yet been verified. Different models or firmware generations may behave differently.

If you confirm another model or firmware version, a report would be useful.

## Password algorithm

The tested firmware normalizes the MAC address as lowercase, colon-separated text and calculates:

```text
MD5(lowercase_mac + "::innomediaadmin")
```

The first eight bytes of the raw MD5 digest are used. Each byte is masked with `0x1f`, then mapped to either `a-z` or `3-8` to produce the eight-character password.

The JavaScript in `innomedia_password_generator.html` implements that calculation directly.

## Important

Use this only with devices you own or have permission to administer.

No InnoMedia firmware, private keys, certificates, or copyrighted firmware code are included in this project.

## License

Copyright (C) 2026 WormFood.

This project is licensed under the GNU General Public License version 3. See `LICENSE`.
