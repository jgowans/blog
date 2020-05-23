---
layout: post
title:  "An Assortment of Linux Debug Options"
date:   2020-04-17 00:00:00 +0000
categories: tutorial
---

This is a collection of some of the interesting or useful debug kernel config options I've stumbled across. Mostly I don't have an actual application for them; I just find they give some visibility into how parts of the kernel works or how things are structured.

## Basics

There are a few things we'll generall want on.


```
DEBUG_KERNEL=y
DEBUG_FS=y
```

## Page tables

```
PTDUMP_DEBUGFS=y
```

Creates the directory `/sys/kernel/debug/page_tables`

The `current_kernel` file in there shows the virtual regions that exist in kernel space.
That's the practical reealization of the theoretical memory map: [https://github.com/torvalds/linux/blob/master/Documentation/x86/x86_64/mm.rst](https://github.com/torvalds/linux/blob/master/Documentation/x86/x86_64/mm.rst).

Compare the layout with 4 or 5 level page tables.

```
CONFIG_X86_5LEVEL=y
CONFIG_PGTABLE_LEVELS=5
```

## IOMMU

```
IOMMU_DEBUGFS=y
INTEL_IOMMU_DEBUGFS=y
```

Creates: `/sys/kernel/debug/iommu/intel`
As well as seeing IOMMU registers it has the `domain_translation_struct` file which seems to walk and print the entire IOMMU page table. From IOVA_PFN through PML5E/PML4E/PDPE/PDE/PTE for every IOVA, for each PCI device.

## Sl(a|o|u)b

## Bug hunting

If there are bugs, make sure we'll notice them.

```
CONFIG_BUG=y
CONFIG_GENERIC_BUG=y
```

```
KALLSYMS
```

## gdb

```
DEBUG_INFO
```

```
KALLSYMS_ALL
```
