---
title: Creating a Bootable Windows USB on Linux
date: 2024-12-18 09:00:00 +0100
published: true
categories: [Post, Linux]
tags: [linux, file-system, sys-admin, hybrid-iso, command-line, windows, bootable-usb]     # TAG names should always be lowercase
---


Creating a bootable USB drive is an essential skill for IT professionals and enthusiasts alike. In this post, we'll walk you through how to create a Windows bootable USB drive on a Linux machine using only terminal commands. Whether you're setting up a new system, troubleshooting an existing one, or simply need a reliable installer on the go, this guide has you covered.

We'll break down each step, from preparing and partitioning your USB drive to writing the Windows ISO to the drive, all with straightforward terminal commands.

## Linux’s ISOHybrid vs. Windows ISO

When creating a bootable USB drive, understanding the differences between Linux ISOHybrid images and Windows ISO images is crucial. While both serve as installation mediums, they differ in their design, functionality, and use cases. These differences impact how the images interact with USB drives and how tools like `dd` or specialized utilities are used to make them bootable.

This distinction becomes especially important when creating a bootable Windows USB from a Linux environment, as the process often requires additional steps compared to working with a Linux ISOHybrid. Let's dive into the details to better understand these formats and their implications for creating bootable USB drives.

This context sets the stage for understanding the specific challenges and solutions when working with a Windows ISO on Linux, and how to identify them.

### What Is a Hybrid ISO?

A hybrid ISO is a special type of ISO image that can function as both a standard optical disk image and a bootable image for devices like USB drives. Traditionally, ISO images were designed to be burned onto CDs or DVDs and used as bootable media. Hybrid ISOs extend this functionality, allowing the same image to also be written directly to a USB drive and used as bootable media without additional modification.

This dual capability is achieved by including both a traditional ISO9660 filesystem for CD/DVD compatibility and additional structures (like a Master Boot Record, or MBR) that make the image bootable when written to a USB drive.

### How to Identify a Hybrid ISO

You can determine whether an ISO is a hybrid using the following methods:

1. Using `file` Command

   The file command can give you information about the ISO’s structure:

   ```bash
      file path/to/image.iso
   ```

   For a hybrid ISO, the output will typically mention the presence of "(DOS/MBR boot sector)".

2. Inspect the MBR with `fdisk`

   Hybrid ISOs include an MBR to support USB booting. You can inspect this with `fdisk`:

   ```bash
      fdisk -l path/to/image.iso
   ```

      If you see partition information (e.g., a single bootable partition), the ISO is likely hybrid.
      A non-hybrid ISO will not show partition details.

3. Check for El Torito Boot Record

   Hybrid ISOs use the El Torito[^El-Torito] specification for booting. You can verify this with the xorriso tool:

   ```bash
      xorriso -indev path/to/image.iso
   ```

   The output will indicate if the ISO contains a boot loader and a partition table under the `Boot record` entry.

## What You’ll Need

Before diving into the process, make sure you have:

1. **A Windows ISO file**: Download the official ISO from the [Microsoft website](https://www.microsoft.com/software-download/windows).
2. **A USB drive**: Ensure it has at least 8GB of space and back up any important data, as the process will erase all contents.
3. **Linux terminal access**: All commands will be executed from the terminal.

---

## Step 1: Identify Your USB Drive

Insert your USB drive and open the terminal. Use the `lsblk` command to identify the device name of your USB drive:

```bash
lsblk
```

In the output, locate your USB drive, usually named something like `/dev/sdX`, where `X` represents the drive letter. Be cautious to note the correct device.

---

## Step 2: Partition USB Drive

Before formatting, create the necessary partitions on the USB drive. Use the `fdisk` tool to manage partitions:

> Unmount any partitions on the USB drive to avoid errors during partitioning and formatting:
>
> ```bash
> sudo umount /dev/sdX*
> ```
>
> Replace `/dev/sdX*` with the appropriate partition(s) from your USB drive.
{: .prompt-tip }

1. Start `fdisk`:

   ```bash
   sudo fdisk /dev/sdX
   ```

2. Inside the `fdisk` prompt, create a new partition table:
   - Press `g` to create a new `GPT` partition table.
   - Press `n` to create a new partition.
   - Choose `p` for a primary partition.
   - Create the first partition only leaving some spare room for the boot partition. Around a Meg should be enough.
   - Repeat the steps for the second partition taking the remaining space.

   > In Windows environments, especially when dealing with USB drives, the operating system typically recognizes only the first partition on removable drives. This means that if your USB drive is configured with multiple partitions, Windows will usually mount and assign a drive letter only to the first primary partition. Consequently, to ensure that the data partition is accessible in Windows, it should be the first partition on the USB drive. The boot partition, containing system or boot files, should be placed as the second partition.
   {: .prompt-info }

3. Set the the file system type to `Microsoft basic data`:
   - Press `t` and select the file system type number `11` for both partitions.

4. Set the bootable flag for the partition:
   - Press `x` for extra functionality.
   - Press `A` and select the **bootable** partition.

5. Write the changes and exit:
   - Press `w` to write the changes and exit `fdisk`.

---

## Step 3: Format USB Drive

Use NTFS for the data partition and FAT32 for the boot partition:

**For NTFS:**

```bash
   sudo mkfs.ntfs -f /dev/sdX1
```

**For FAT32:**

```bash
   sudo mkfs.vfat -F 32 /dev/sdX1
```

Replace `/dev/sdX1` with the correct partition created in the previous step.

---

## Step 4: Copy Windows ISO files to the USB Drive

Mount the ISO File and the drive data partition and copy the contents of the ISO to the USB:

1. Mount the ISO File:

   Open a terminal and create a mount points:

   ```bash
      sudo mkdir /mnt/iso
      sudo mkdir /mnt/drive
   ```

2. Mount the ISO:

   ```bash
      sudo mount -o loop /path/to/your.iso /mnt/iso
      sudo mount /dev/sdX1 /mnt/drive
   ```

   Replace `/path/to/your.iso` with the actual path to your ISO file, and `/dev/sdX1` with the correct partition created in the previous step.

3. Copy the Files:

   Use the `cp` command to copy files:

   ```bash
      sudo cp -r /mnt/iso/* /path/to/destination/folder
   ```

   Replace `/path/to/destination/folder` with your target directory, in this case `/mnt/drive`.

   > This can take a while, around 10 min depending of your USB speeds.
   {: .prompt-info }

4. Unmount the ISO:

   After copying, unmount the ISO:

   ```bash
      sudo umount /mnt/iso
   ```

>Remove the mount point if no longer needed:
>
> ```bash
>    sudo rmdir /mnt/iso
>   ```
>
{: .prompt-info }

---

## Step 5: Write Rufus boot image to drive

For it first of all we will need to download the image from the Rufus[^Rufus] repository.

```bash
wget https://github.com/pbatard/rufus/raw/master/res/uefi/uefi-ntfs.img
```

Use the powerful `dd` command to write the Windows ISO directly to the USB drive:

```bash
sudo dd if=/path/to/windows.iso of=/dev/sdX bs=1M status=progress
```

- Replace `/path/to/windows.iso` with the actual path to your Windows ISO file.
- Replace `/dev/sdX` with your USB device name.

> Double-check the device name to avoid overwriting important drives.
{: .prompt-warning }

---

## Step 6: Install boot loader

Use `gub2-install` to install boot

```bash
sudo grub2-install --target=i386-pc --boot-directory=/path/to/windows_part --force /dev/sdX
```

- Replace `/path/to/windows_part` with the actual path to your USB Windows data partition mount point.
- Replace `/dev/sdX` with your USB device name.

> Double-check the device name to avoid overwriting important drives.
{: .prompt-warning }

---

## Step 7: Test the Bootable USB

Plug the USB drive into your target system and boot from it. You may need to adjust your BIOS/UEFI settings to prioritize USB boot and disable secure boot if necessary.

---

## Troubleshooting Common Issues

- **Bootloader Problems:** If the USB fails to boot, the ISO might require additional setup.
- **Permission Denied Errors:** Always run the commands with `sudo` to ensure you have the necessary permissions.
- **Corrupted ISO:** Verify your ISO file’s integrity with a checksum:

  ```bash
  sha256sum /path/to/windows.iso
  ```

  Compare the result with the official checksum.

---

## Conclusion

With these simple terminal commands, you can create a Windows bootable USB drive right from your Linux machine. Whether for a new installation or recovery, having a bootable USB prepared ensures you're always ready for the task at hand.

Happy booting!

## References

[^El-Torito]: <https://en.wikipedia.org/wiki/ISO_9660#El_Torito>
[^Rufus]: <https://rufus.ie/en/>
