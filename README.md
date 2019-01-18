cryptboot
=========

Encrypted boot partition manager with UEFI Secure Boot support

Description
-----------

With encrypted boot partition, nobody can see or modify your kernel image or initramfs.
GRUB boot loader supports booting from encrypted boot partition, but you would be
still vulnerable to [Evil Maid](https://www.schneier.com/blog/archives/2009/10/evil_maid_attac.html)
attacks.

One possible solution is to use UEFI Secure Boot. Get rid of preloaded Secure Boot keys
(you really don't want to trust Microsoft and OEM), enroll your own Secure Boot keys
and sign GRUB boot loader with your keys. Evil maid would be unable to boot
modified boot loader (not signed by your keys) and whole attack is prevented.

`cryptboot` simply makes this easy and manageable.

Requirements
------------

- Linux (x86_64)
- UEFI firmware with enabled Secure Boot
- separate `/boot` partition encrypted with LUKS
- cryptsetup
- openssl
- efitools
- sbsigntools
- efibootmgr
- grub (grub-efi on Debian based distributions)

Installation
------------

1. Install your favorite Linux distribution with separate `/boot` partition encrypted with LUKS.
   Refer to your distributions documentation, there is e.g. guide for Arch Linux:

   [Encrypted boot partition (GRUB)](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Encrypted_boot_partition_.28GRUB.29)

2. Boot into UEFI firmware setup utility (frequently but incorrectly referred to as "BIOS"),
   enable *Secure Boot* and clear all preloaded Secure Boot keys (Microsoft and OEM).
   By clearing all Secure Boot keys, you will enter into *Setup Mode*
   (so you can enroll your own Secure Boot keys later).

   You must also set your UEFI firmware *supervisor password*, so nobody
   can simply boot into UEFI setup utility and turn off Secure Boot.

3. Boot into your Linux distribution and mount `/boot` partition and EFI System partition:

        cryptboot mount

4. Generate your new UEFI Secure Boot keys:

        cryptboot-efikeys create

5. Enroll your newly generated UEFI Secure Boot keys into UEFI firmware:

        cryptboot-efikeys enroll

6. Update GRUB boot loader (it will be automatically signed with your new UEFI Secure Boot keys):

        cryptboot update-grub

7. Unmount `/boot` partition and EFI System partition:

        cryptboot umount
        
8. Optional: Install cryptboot-grub-warning script to raise an error if running `grub-install` directly instead of `cryptboot update-grub`:

        ln -s /etc/cryptboot-grub-warning /usr/local/bin/grub-install

9. Reboot your system, you should be completely secured against evil maid attacks from now on!

Usage
-----

After installation, usage of `cryptboot` is as simple as running:

    cryptboot upgrade

This will mount `/boot` partition and EFI System partition, properly upgrade your system
with distributions package manager, update and sign GRUB boot loader and finally
unmount `/boot` partition and EFI System partition.


Help
----

**cryptboot**

    Usage: cryptboot {mount|umount|update-grub|upgrade}
    
    Encrypted boot partition manager with UEFI Secure Boot support
    
    Commands:
        mount        Unlock and mount your encrypted boot partition and EFI System partition
        umount       Unmount and lock your encrypted boot partition and EFI System partition
        update-grub  Update GRUB2 boot loader and sign it with your UEFI Secure Boot keys
        upgrade      Mount, upgrade system with package manager, update boot loader and unmount

**cryptboot-efikeys**

    Usage: cryptboot-efikeys {create,enroll,sign,verify,list} [file-to-sign-or-verify]
    
    Manage UEFI Secure Boot keys
    
    Commands:
        create  Generate new UEFI Secure Boot keys
        enroll  Enroll new UEFI Secure Boot keys to your UEFI firmware
                (you have to clear old keys in your UEFI firmware setup utility first)
        sign    Sign EFI boot image file with your UEFI Secure Boot keys
        verify  Verify signature of EFI boot image file with your UEFI Secure Boot keys
        list    List all UEFI Secure Boot keys enrolled in your UEFI firmware
        status  Check if UEFI Secure Boot is active or inactive

**Default configuration (`/etc/cryptboot.conf`)**

    # Encrypted boot device name (/dev/mapper/$BOOT_CRYPT_NAME)
    # (have to be specified in /etc/crypttab)
    BOOT_CRYPT_NAME="cryptboot"
    
    # Boot partition mount point (have to be specified in /etc/fstab)
    BOOT_DIR="/boot"
    
    # EFI System partition mount point (have to be specified in /etc/fstab)
    EFI_DIR="/boot/efi"
    
    # Default boot loader (only GRUB is supported for now)
    BOOT_LOADER="GRUB"
    
    # Boot entry in UEFI Boot Manager (if using GRUB boot loader)
    EFI_ID_GRUB="GRUB"
    
    # Path to GRUB boot loader EFI file (relative to EFI System partition)
    EFI_PATH_GRUB="EFI/grub/grubx64.efi"
    
    # UEFI Secure Boot keys directory
    EFI_KEYS_DIR="/boot/efikeys"
    
    # Command run to upgrade system packages
    PKG_UPGRADE_CMD="pacman -Syu"

Limitations
-----------

- If there is backdoor in your UEFI firmware, you are out of luck. It is *GAME OVER*.

  Old laptops unfortunately regularly had backdoors in BIOS:

  [BIOS Password Backdoors in Laptops](https://dogber1.blogspot.cz/2009/05/table-of-reverse-engineered-bios.html)

  New laptops (as of 2016) should be hopefully more secure, but I am only sure about
  Lenovo ThinkPads (there are no known backdoor passwords and Lenovo is reportedly
  replacing whole system board if user forgets his supervisor password).

- You should never use same UEFI firmware *supervisor password* as your encryption password,
  because on some old laptops, supervisor password could be recovered as plaintext
  from EEPROM chip.
  
  New Lenovo ThinkPads (T440, T450, T540, X1 Carbon gen2/3, X240, X250, W540, W541, W550
  and newer models) should be safe, see e.g. this presentation:

  [ThinkPad BIOS Password Design for UEFI](http://monitor.espec.ws/files/lewnovo_password_399.pdf)

- Attacker can also directly reflash your UEFI firmware with his own modified evil firmware,
  but this can be prevented by physical means (e.g. epoxy resin ;-)).

  There are also [procedures](http://www.allservice.ro/forum/viewtopic.php?t=3044) how to reset
  supervisor password even on modern ThinkPads with SPI serial flash programmer. Again, you can
  use physical means for prevention.

- If you have encrypted boot partition, you can't easily use another TPM-based
  trusted / verified boot solution like [tpmtotp](https://github.com/mjg59/tpmtotp)
  or [anti-evil-maid](https://github.com/QubesOS/qubes-antievilmaid/tree/master/anti-evil-maid).

  This is because these solutions rely on running code from initramfs before you enter
  decryption password. But if you have encrypted boot partition, you have to enter decryption
  password before loading initramfs, so it would be already too late for these solutions to
  have any effect (evil firmware / boot loader will already have your password at that point).

  This can be fixed by implementing TPM support and `tpmtotp` or `anti-evil-maid` like
  functionality directly in GRUB boot loader (but current [TrustedGRUB2](https://github.com/Rohde-Schwarz-Cybersecurity/TrustedGRUB2)
  doesn't even support UEFI yet).

  The question is if this is really needed? If you don't trust UEFI firmware, why should you
  trust TPM? But nevertheless it would be nice to have double-check against evil maids.
