+++
title = 'SD Cards'
date = 2025-04-18T18:26:52-05:00
draft = false 
+++

## Why have an EMMC at all

When we started with TAOS we were trying to run on x86 laptops. 
The one we had focused on was a laptop that had eMMC storage, so 
since eMMC is basically a fancy SD card, The first thing I worked on was 
an SD card driver that could read and write blocks to unblock the file 
system people. 

## Setting Up the driver