# Lizard Architecture🦎

This document serves as my architecture brainstorming.

# Compiling into a mmap file

On Linux I want to compile directly into a memfd file first using HUGE PAGE support compiling into 1 gig pages then trying 2MB if 1 gig fails fianlly into your usual 4KB paged memory. The reason for this is running 2MB/1 gig page aware code requires a lot of custom tooling - LD prehacks, writing your own ELF loader, etc. With current compilerers insistence on having text, data, etc being seperated it means 1 gig page aware programs at compile time is tough.
