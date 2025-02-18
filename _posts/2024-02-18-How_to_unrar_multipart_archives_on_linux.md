---
title: How to Extract Multi-Part RAR Files on Linux.
date: 2024-02-18 20:00:00 +0100
categories: [How-To, Archive, RAR] # Posts, Tutorials, Guides, How-To Articles, Opinion Pieces, News & Updates, Comparisons, Reviews, Case Studies, Cheat Sheets & Reference Guides
tags: [linux, rar, compression, 7zip, archive] # TAG names should always be lowercase
---
  
Extracting multi-part RAR files on Linux can be a frustrating task, primarily because RAR is a proprietary format that is not natively supported by many Linux distributions. This often results in compatibility issues, especially when using open-source alternatives like `unrar-free`[^unrar-free], which struggles to properly handle multi-part archives.

## What is RAR?

The RAR[^rar-wiki] file format, developed by Eugene Roshal, is a proprietary compression format owned by RARLab.[^rarlab] Unlike open-source alternatives such as ZIP or tar.gz, RAR is not freely licensed, making it unavailable in many Linux distributions by default. The official `unrar` utility for Linux is distributed under a restrictive license that allows only extraction, not compression. Users needing full functionality must purchase a licensed version of WinRAR. Due to these restrictions, Linux users often face issues when working with multi-part RAR archives.

## Methods to Extract Multi-Part RAR Files on Linux

Below are three methods to extract multi-part RAR files, ranked by simplicity and the number of additional tools required.

### 1. Merging Multi-Part RAR Files into a Single Archive

Since the open-source `unrar-free` tool can handle single RAR files but struggles with multi-part archives, one workaround is to concatenate all parts into a single file before extraction.

> Although the binary or executable created by `unrar-free` is named `unrar`, it should not be confused with the `unrar` executable from WinRAR (RARLab).
{: .prompt-info}

```bash
cat archive.part1.rar archive.part2.rar archive.part3.rar > complete_archive.rar
unrar x complete_archive.rar
```
{: .nolineno }

>This method works well for some cases, but it may not always be reliable, particularly for encrypted or highly compressed archives.
{: .prompt-warning }

### 2. Using 7-Zip for Extraction

7-Zip[^7zip] (`7zz`) provides better support for multi-part RAR archives compared to open-source `unrar-free`. While 7-Zip does not support creating RAR files, it can extract them without issue.

First, install 7-Zip from your package manager or download it from the [7-Zip website](https://www.7-zip.org/download.html).

```bash
7zz x archive.part1.rar
```
{: .nolineno }

>7-Zip is released under the GNU Lesser General Public License (LGPL) v2.1 but includes unRAR code, which has additional licensing restrictions. Because of this, some Linux distributions (like Fedora and Debian) exclude RAR support from their official repositories.
{: .prompt-tip }

### 3. Using WinRAR

WinRAR provides the most reliable method for handling RAR files, as it is the official software for the format. However, it is proprietary software that requires a license for long-term use.

You can download WinRAR from [RARLab’s official site](https://www.rarlab.com/download.htm). After installation, you can use either `rar` or `unrar` to extract the files:

```bash
unrar e archive.part1.rar
rar e archive.part1.rar # Both 'rar' and 'unrar' can extract RAR archives
```
{: .nolineno }

> While WinRAR offers the best compatibility, its restrictive license makes it the least desirable option for open-source users.
{: .prompt-tip }

## Conclusion

Whenever possible, use open-source compression formats such as `.zip` for cross-platform compatibility or `.tar.gz` for Linux systems. This avoids licensing restrictions and ensures that files can be easily extracted on any operating system.

## References

[^rar-wiki]: <https://en.wikipedia.org/wiki/RAR_(file_format)>
[^rarlab]: <https://www.rarlab.com/>
[^unrar-free]: <https://gitlab.com/bgermann/unrar-free>
[^7zip]: <https://www.7-zip.org/>
