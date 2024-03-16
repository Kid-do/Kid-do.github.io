---
title: "Installing a Second SSD in the Lenovo ThinkPad T480s"
header:
  image: /assets/images/2020-12-29.jpg
  caption: "Photo credit: [**Myself**](/)"
  teaser: /assets/images/2020-12-29.jpg
categories:
  - blog
---

tl;dr: The T480s supports booting from a second NVMe SSD in the WWAN slot.

Since I purchased a ThinkPad T480s I have wanted to install a second SSD to dual boot Linux and Windows. I found many reports online about this being supported on a T480, but mixed reports about the T480s. I decided to give it a go, here is a quick guide on how I did it.

First, you need to ensure you get an NVMe drive in the M.2 2242 form factor. I chose a WD SN520 256GB. SATA drives will not work. Disable the internal battery through the laptop's BIOS, then unscrew and remove the bottom cover. The WWAN slot is intended for use with Mobile Broadband Cards, but we will be installing our SSD here. Peel off the sticker holding the antenna cables and move them out of the way. Install the SSD in its slot (screw should be provided) and screw the bottom cover back on. I then cloned my Windows install from the original 512GB SSD to the new, lower capacity SSD using Macrium Reflect before installing my chosen Linux distro onto the original SSD.

The final result is having a dual boot setup on a laptop that doesn't 'officially' support two SSDs. I can set the default OS in the BIOS, and if I want to boot into Windows, I press F12 at boot to change the boot device.
