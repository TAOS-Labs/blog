+++
title = 'TAOS Overview'
date = 2025-04-18T18:26:52-05:00
draft = true
+++
## TAOS - Totally Awesome Operating System

Is a distributed operating system designed by University of Texas Austin Computer Science students as a project for CS 378: Multicore Operating Systems. Designed for x86 architecture this system is designed to support processes spawning processes on linked remote hosts running TAOS. The system abstracts the distributed nature of the processes from the end-user without requiring any action on their part. To this end TAOS will automatically handle the distribution of memory and the filesystem, while also deciding when to route IPC to a remote host.