---
title: Where to manually install your programs on Linux.
date: 2024-12-18 11:00:00 +0100
categories: [Post, Linux]
tags: [linux, file-system, sys-admin]     # TAG names should always be lowercase
---

# **Where to Install Applications and Place Binaries on Linux**

When manually installing software, understanding where to place binaries and supporting files is essential to keeping your Linux system clean, organized, and maintainable. The Linux file system follows standards defined by the **Filesystem Hierarchy Standard (FHS)** and additional conventions like the **XDG Base Directory Specification**. But where do all these directories come from, and why are they used this way? In this blog post, we'll break down the major directories for binaries, libraries, and application files, their history, and best practices for manual installation.

---

## **A Brief History of the Linux Filesystem**

The layout of the modern **Linux file system**, as often happens in software, is a combination of smart ways to circumvent hardware limitations from its early days, carried over time in the pursuit of backward compatibility, and the implementation of new standards to achieve improvements and simplification.

The file system has its roots in the creation of **UNIX** in 1969. At that time, only some directories were created to maintain the organization of different files, such as:

- **`/bin`**: Essential system binaries required for booting and single-user mode.
- **`/sbin`**: Critical tools for system repair and configuration.
- **`/lib`**: Libraries for the binaries in `/bin` and `/sbin`.

In 1971, when disk space was limited and expensive, the PDP-11[^pdp-11], for example, used two RK05[^rk05] disk packs (2.5MB each). As the operating system grew too large for the first disk (used as the root filesystem), it overflowed into the second disk, where user home directories were stored, leading to the creation of the `/usr` mount point. OS directories were duplicated there, creating the paths `/usr/bin` and `/usr/sbin`. Since some binaries in `/usr` wouldn’t be accessible until the drive was mounted, they segmented the binaries into those needed at boot time (`/bin`) and those not essential during boot (`/usr/bin`).

- **`/usr`**: Originally stood for "Unix System Resources", where user applications and libraries were stored.

Over time, as systems grew more complex, new directories were introduced to house software installed **outside of the base operating system**, leading to directories like `/usr/local`, `/opt`, and the adoption of the **FHS**.

The **Filesystem Hierarchy Standard (FHS)**, maintained by the Linux Foundation, formalized these conventions. It defines where software, binaries, and libraries should go, ensuring consistency across distributions.

---

## **Common Directories for Binaries and Applications**

Here's a breakdown of the directories where binaries and applications can be installed, including their intended purposes:

### **1. /bin and /usr/bin**

- **Purpose**:
  - `/bin`: Essential binaries needed for the system to boot and run in single-user mode.
  - `/usr/bin`: Non-essential user binaries.
- **History**:
  - Early Unix systems only had `/bin`. Later, as systems grew larger, `/usr/bin` was introduced to store less critical programs to reduce the size of the root filesystem.
  - Nowadays, these two directories are often consolidated into one, as larger and more affordable drives have become mainstream. We also see that `/bin` is typically a symbolic link to `/usr/bin`, as a fix to maintain backward compatibility.
- **Use Today**:
  - Managed by the package manager (e.g., `dnf`, `apt`) and should **not** be used for manual installations.

> **Example binaries**:
> - `/bin/ls` (list directory contents)
> - `/usr/bin/vim` (text editor)
{: .prompt-info }

---

### **2. /sbin and /usr/sbin**

- **Purpose**:
  - `/sbin`: System binaries required for system administration (e.g., `fsck`, `iptables`).
  - `/usr/sbin`: Non-essential system administration tools.
- **History**:
  - As happens with `/bin`, we can find `/sbin` and `/usr/sbin` consolidated as well.
- **Who Uses It**: These directories contain programs for the **root user** or system administrators.
- **Best Practice**: Do not place manual binaries here. Use `/usr/local/bin` instead.

Example binaries:

    /sbin/fsck (file system check)
    /usr/sbin/nginx (web server)

> **Example binaries**:
> - `/sbin/fsck` (file system check)
> - `/sbin/lvm` (logical volume manager)
{: .prompt-info }

---

### **3. /usr/local/bin**

- **Purpose**: Binaries installed manually by the system administrator, separate from package-managed files.
- **Why It Exists**: In the early days, `/usr` was often mounted read-only, so `/usr/local` was created as a writable space for locally installed software.
- **Best Practice**: Use `/usr/local/bin` for system-wide, manually installed binaries that are available to all users.

> **Example**:
> ```bash
> sudo cp myprogram /usr/local/bin/
> ```
{: .prompt-info }

> - **Libraries** for these binaries should go into `/usr/local/lib`.
{: .prompt-tip }

---

### **4. /opt**

- **Purpose**: A directory for optional, self-contained, or third-party software packages.
- **Why It Exists**:
  - Introduced by the **FHS** to provide a clean location for large software packages that do not conform to the `/usr` structure.
  - Typically used for proprietary or precompiled software.
- **Structure**: Each application gets its own subdirectory:
  ```
  /opt/<appname>/
      bin/   # Binaries
      lib/   # Libraries
      share/ # Shared data
  ```
- **Best Practice**: Use `/opt` for large, self-contained applications (e.g., Prometheus, Apache Spark).

> **Example**:
> ```bash
> sudo mkdir -p /opt/myapp/bin
> sudo cp myapp /opt/myapp/bin/
> sudo ln -s /opt/myapp/bin/myapp /usr/local/bin/myapp
> ```
{: .prompt-info }

> By creating a symbolic link in `/usr/local/bin`, you make the program executable without changing your `PATH`.
{: .prompt-tip }

---

### **5. ~/.local/bin**

- **Purpose**: User-specific binaries for manual installation without requiring root privileges.
- **Why It Exists**:
  - The **XDG Base Directory Specification** (freedesktop.org) introduced `~/.local` as a location for user-specific software and data.
- **Best Practice**: Use `~/.local/bin` for binaries you install just for your user account.

> **Example**:
> ```bash
> mkdir -p ~/.local/bin
> cp myprogram ~/.local/bin/
> ```
{: .prompt-info }

> If needed add the `PATH` modification to your shell configuration (e.g., `~/.bashrc`) to make it permanent.
{: .prompt-tip }

---

### **6. /usr/games and /usr/local/games**

- **Purpose**: Directories for game binaries.
- **Why It Exists**: Historically, Unix systems kept games separate from other programs to make it easier to exclude them from production systems.
- **Use Today**: Rarely used outside specific distributions or systems.

---

## **Where to Install Libraries?**

Binaries often require libraries (shared or static) to function. Here are the corresponding directories:

1. **System Libraries**:
   - `/lib`: Libraries for binaries in `/bin` and `/sbin`.
   - `/usr/lib`: Libraries for binaries in `/usr/bin` and `/usr/sbin`.

2. **Manual/Custom Libraries**:
   - `/usr/local/lib`: Libraries for binaries installed in `/usr/local/bin`.
   - `$HOME/.local/lib`: User-specific libraries for binaries in `$HOME/.local/bin`.

3. **/opt/<appname>/lib**: Self-contained libraries for software in `/opt`.

> To ensure the system can locate custom libraries, you may need to update the **dynamic > linker** with:
> ```bash
> sudo ldconfig
> ```
> Or set the `LD_LIBRARY_PATH` environment variable:
> ```bash
> export LD_LIBRARY_PATH="$HOME/.local/lib:$LD_LIBRARY_PATH"
> ```
{: .prompt-warning }

---

## **Summary of Best Practices**

| **Directory**           | **Purpose**                               | **When to Use**                           |
|--------------------------|-------------------------------------------|-------------------------------------------|
| `/usr/local/bin`         | System-wide manually installed binaries  | Use for clean manual installations.       |
| `/opt/<appname>`         | Self-contained third-party software      | Use for large, standalone applications.   |
| `$HOME/.local/bin`       | User-specific binaries                   | Use for user-only manual installations.   |
| `/usr/local/lib`         | Libraries for manual binaries            | Use alongside `/usr/local/bin`.           |
| `$HOME/.local/lib`       | User-specific libraries                  | Use alongside `$HOME/.local/bin`.         |

---

## **Final Thoughts**

The Linux filesystem may seem complex at first, but its organization is rooted in decades of history and thoughtful design. Whether you're installing a simple script or a large third-party application, understanding the purpose of directories like `/usr/local`, `/opt`, and `~/.local` will help you keep your system clean and maintainable.

Keep your distro clean 🐧.


## References
[^pdp-11]: <https://en.wikipedia.org/wiki/PDP-11>
[^rk05]: <https://en.wikipedia.org/wiki/RK05>