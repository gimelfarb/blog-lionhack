Lenovo Yoga Pro 2 Partition Cleanup
===================================

How to cleanup default factory partitions and reclaim some previous space on the SSD drive on your new Lenovo Yoga Pro 2.

Recently I got the [Yoga Pro 2](http://shop.lenovo.com/ca/en/laptops/ideapad/yoga/yoga-2-pro/), which is [one of the best ultrabooks][1] on the market right now, and to opted for the 256GB SSD model, which is a much better value for money than the top-spec'ed 512GB. 256GB is not that much space by today's standards, and I was already over that on my previous laptop, mainly due to the large library of photos. 

[1]: http://www.forbes.com/sites/davealtavilla/2013/12/20/top-ultrabooks-of-the-season-for-last-minute-shoppers/

Since I already felt tight for getting only 256GB, it was surprising to see that Windows was showing only **218GB available**!! That's not right - already down almost 40 from the envisioned number. A saga to reclaim what's rightfully mine had begun...

### 256 GB doesn't mean what you think

Firstly, the disk itself reports **238.35 GB** in total. Not a surprise - I should have [known better][2] given my background. Advertised **256 GB** really means 256 billion bytes, using powers of 10 for the "G". In computer terms, "G" (giga) or "billion" refers to powers of 2:

[2]: http://www.glyphtech.com/support/diskcapacity.php

<table>
    <thead><th>Term</th><th>Computer Meaning</th><th>Hard Disk Marketing</th><th>% Diff</th></thead>
    <tbody>
        <tr><td>KB</td><td>2<sup>10</sup> = 1,024 bytes</td><td>1,000 bytes</td><td>2.4%</td></tr>
        <tr><td>MB</td><td>2<sup>20</sup> = 1,048,576 bytes</td><td>1,000,000 bytes</td><td>4.8%</td></tr>
        <tr><td>GB</td><td>2<sup>30</sup> = 1,073,741,824 bytes</td><td>1,000,000,000 bytes</td><td>7.4%</td></tr>
    </tbody>
</table>

So, the advertised capacity of 256 GB actually means:

> 256 "GB" = 256 x 10<sup>9</sup> / 2<sup>30</sup> = **238.4 GB** 


### Understanding Lenovo Yoga Pro 2 Partitions

Looking at the Disk Manager, here's the layout you see:

![Disk Layout](http://static.lionhack.com/images/2013-12-25-lenovo-yoga-2-pro-partitions/disk-layout.png "Factory disk layout")

**NOTE:** To open Disk Manager on Windows 8, hit `WinKey+X`, then choose "Disk Management".

In fact, there is one partition that is not shown here, because it is hidden. Running _diskpart_ on Command Prompt reveals it:

![Diskpart Output](http://static.lionhack.com/images/2013-12-25-lenovo-yoga-2-pro-partitions/diskpart-partitions.png "All factory disk partitions")

So what are all these partitions eating up the space? In the past, [there have been complaints][3] about Lenovo messy disk partitioning. Shouldn't it have been solved? Well, it is in fact much better compared to previous Yoga edition. But there are still 7 partitions on the disk - are they really necessary!?

[3]: http://www.zdnet.com/lenovo-cleans-up-its-incredibly-messy-yoga-13-disk-layout-7000008379/

Well, some of them are. The answer comes from Microsoft, who publish a [recommended disk layout for Windows 8 UEFI-based installations][4]. More information about GPT disks on Windows can be found [here][5] and [here][6]. Here's a short description of each partition:

  * Partition 1 - Recovery (**WINRE_DRV**) - **1000MB**
    * _Windows RE bootable (for Recovery Mode)_
    * _Used when booting into Windows Recovery (Win RE) environment_
  * Partition 2 - System (**SYSTEM_DRV**) - **260MB**
    * _System UEFI bootable (EFI/Windows boot menu)_
    * _UEFI boots this partition, it contains Windows NTLDR, HAL, Boot.txt, and some drivers (Windows will not boot without this)_
  * Partition 3 - OEM (**LRS_ESP**) - **1000MB**
    * _Lenovo Recovery System (EFI bootable)_
    * _Lenovo [One Key Recovery (OKR)][7] button boots into this partition, which allows to do factory restore_ 
  * Partition 4 - Reserved (**MSR**) - **128MB**
    * _Reserved Microsoft partition for GPT-based disks_
    * _[Must exist and must be 128MB][6], used by Windows when moving/changing partitions through Disk Manager_
  * Partition 5 - Primary (**Windows8_OS**) - **218GB**
    * _Main C: drive - contains Windows, installed programs, etc_
  * Partition 6 - Primary (**LENOVO**) - **4GB**
    * _Lenovo D: drive - contains mainly drivers and installers for some bundled apps_
  * Partition 7 - Recovery (**PBR_DRV**) - **13GB** 
    * Lenovo factory reset image
    * Used by One Key Recovery system to reset laptop to factory condition

[4]: http://technet.microsoft.com/en-us/library/dd744301(v=ws.10).aspx
[5]: http://technet.microsoft.com/en-us/library/dd799232(v=ws.10).aspx
[6]: http://msdn.microsoft.com/en-us/library/windows/hardware/gg463525.aspx#X-201104111922443
[7]: http://www.lenovo.com/shop/americas/content/user_guides/yoga2_ug_en.pdf

What can be removed from this list? Partitions 1, 2 and 4 are essential for Windows proper operation on UEFI/GPT system. Partition 3 is necessary for OKR button to work, and uses factory reset image from Partition 7 (by default). Partitions 6 & 7 are the ones where we can reclaim some space - and they add up to **17+ GB**!‎ Not bad, and will leave us **99%** (**236 GB**) of the total disk space for C: drive.

### Creating USB Recovery Drive (Windows 8.1)

Before Partition 7 (PBR_DRV) can be removed, we need to copy the factory reset image somewhere, and be able to use it in case we want to do a factory reset (i.e. when you decide to sell or give away this laptop in the future). Luckily, the way [Lenovo set this up][8] it integrates with Windows 8 recovery mechanisms, and is registered as a standard recovery image.

To verify, you can run `reagentc /info` on Command Prompt:

![Reagentc Output](http://static.lionhack.com/images/2013-12-25-lenovo-yoga-2-pro-partitions/reagentc-info.png "Windows RE settings")

This shows that Lenovo's factory reset image from Partition 7 has been registered as the recovery image with Windows RE settings. This is good news, because you can now follow [these instructions][8] to create a default Windows USB Recovery Drive.

[8]: http://support.lenovo.com/en_US/downloads/detail.page?DocID=HT076024

The instructions are [outlined on Lenovo website][8] and are simple to follow. Note, that since Windows 8.1 you need a USB key to create Recovery Media, **DVD is no longer an available option** (I suspect because factory reset images are pretty large these days and it's an extra complication to have to span multiple DVDs).

In short the steps are:

  1. Go to Control Panel (`WinKey+X`, then select Control Panel), and go to Recovery panel
  2. Select "Create recovery drive" and make sure "Copy the recovery partition from the PC to the recovery drive" checkbox is selected
  3. Follow prompts and insert a USB key when prompted (you need at least the **16 GB** variety)
  4. At the end it will prompt whether you want to delete recovery partition - **DENY this option**, we will delete it manually afterwards

You want to test that the new USB media actually works before proceeding with cleanup. Reboot your PC, and in the boot menu select to boot from USB drive. (To access the boot menu you need to shutdown and [press Novo button][7], the little button next to power button).

If USB key works, you should be able to access Recovery Environment, ready to reset to factory image. Keep this USB key in a safe place for future, when you want to do the reset. Now you can reboot laptop to continue with disk cleanup.

### Backup D: (LENOVO) drive

Backup the files on D: drive. You can either copy them to C: drive (but then they'll continue to take up space), or copy them to external media. You can also copy them to the same Recovery Disk USB that was just created, if there is free space.

### Disk cleanup - removing partitions

Now we are ready to delete Partitions 6 & 7. Start elevated Command Prompt (`WinKey+X`), and start "**_diskpart_**" utility.

**!!! WARNING !!!:** Be **extremely cautions** with _diskpart_. Delete wrong partitions will cause laptop to become unusable, and will require a factory reset. (You did create that Recovery USB Disk now, right?)

Here are commands to remove these partitions (in **diskpart**), verify output after each one to make sure it is doing the right thing:

  1. Deleting Partition 7 (PBR_DRV)
     * _select disk 0_
     * _list part_
     * _select part 7_
     * _detail part_
     * _delete part override_
  2. Deleting Partition 6 (D: drive)
     * _select disk 0_
     * _select part 6_
     * _detail part_
     * _delete part_

**NOTE:** Now Partition 3 (OKR) won't be used for factory reset, the created USB Recovery Disk will be instead. But having this partition still allows Lenovo Recovery boot option to operate, even if it's not very useful. Plus it is difficult to merge that space with Partition 5 (C:) because they are not adjacent, and [other tools](http://en.wikipedia.org/wiki/PartitionMagic) would need to be used - all for a very limited gain (1000 MB). So we won't be doing that.

### Finally, extending C: drive

Now you can go to Disk Management (`WinKey+X`) and see that the last 2 partitions are gone - it should show as "Unallocated space".

  * Right-click on Windows8_OS (C:) partition, and choose Extend Volume
  * Verify it is selecting 18GB of unallocated space (the default)
  * Proceed through the wizard (Next > Next > Finish)

![Disk Layout After](http://static.lionhack.com/images/2013-12-25-lenovo-yoga-2-pro-partitions/disk-layout-after.png "Disk layout after space reclaimed")

You have now successfully extended your space on C: drive! Enjoy the extra freedom, while it lasts. I have already filled mine with more photos!



