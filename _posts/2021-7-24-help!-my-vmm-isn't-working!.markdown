---
layout: post
title: "Reasons why your vmm is broken"
date: 2021-7-24
tags:
  - paging
  - vmm
  - bug
  - x86
  - 64-bit
author: Tim Thompson
avatar: assets/profile.png
category: Bug fixes
---

Hello everyone, welcome to a new blog post. 👋

If you are reading this you are either hopping in to see what I wrote, or your vmm (paging code) isn't working and you don't know why.

Well look no further as I will walk you through some common pitfalls that aren't covered in tutorials.

### Prerequisites
It is assumed that you are using x86_64 (though x86 applies to some concepts here as well), and know about paging. If not, go learn about it and come back to this article. I will not write a paging article as there are some very detailed paging explanations out there. Namely [this one][RUST-PAGING]

### Nothing went wrong, how do I know everything went according to plan?
Two things.

In the qemu emulator open the qemu shell by using the keyboard combo `ctrl+alt+2` or by navigating to `view > compatmonitor0`

1. `info mem` will display the mapped pages in this format: `start_address`-`end_address`-`mappings`-`flags`. If the output is empty then you have no mapped pages.
2. `info registers` will dump the registers. (For more information on how to interpret the output of `info registers` consult the [Understanding qemu logs for debugging][QEMU-LOG-BLOG-POST] blog post). Look at cr3, is the value bogus or 0? If so, something went wrong.

### My kernel does whatever it want's when I try to reload cr3!
You spent all this time setting up your paging routines, reading the manuals, everything is _perfect_ yet you cannot load the page tables into cr3... The processor just exhibits UB (`Undefined Behaviour`).. Why?

There are two gotcha's I know of, one is more subtle than the other.

The first issue could be that you failed to map the pages correctly, in that case you need go over the code that maps your physical addresses to virtual addresses.

The second issue could be that your vmm is perfectly fine! Assuming you are in 64 bit long mode and using a higher half kernel, your pmm might be returning higher half addresses! (Example: `0xffffffff80000000`)

The issue with this is that using such an address for memory allocation would actually fall **2GiB** short of the end of the address space. A direct map could *not* fit in a machine with more than 2GiB of RAM, you simply capped yourself from using the physical RAM! How crazy does that sound?

The solution is simple, instead of using a higher half offset such as `0xffffffff80000000` for memory allocation, you would use `0xffff800000000000`. The latter address is the address of a generic higher half direct map.

Below is an example on how to write pmm_alloc for paging:
{% highlight C %}
#define ADDRESS_BASE 0xffff800000000000ULL

//NOTE: This assumes a bitmap based pmm, change it according to your needs
void *phys_alloc()
{
	uint32_t idx = find_free_block_in_bitmap();

	//Returns a pointer to allocated memory in the higher half address space
	void *ptr = page_frame_allocator_reserve_bit(idx);

	return (void*) ((ptr - ADDRESS_BASE));

	//What NOT to do:
	//return (void*)ptr; <- ptr is a higher half address, remember?
}
{% endhighlight %}

### I'm getting #GP faults and my interrupts aren't working!

Assuming your GDT and IDT worked as expected earlier, it know doesn't seem to work after the vmm.
The issue _could_ lie in your vmm, but before you go searching for a bug that might not exist I advise you to check if you call `gdt_init()` & `idt_init()` *before* `vmm_init()`.

If you load your gdt and idt before mapping page tables your kernel might try to access a gdt or idt entry at the address you said but since you setup virtual memory the address of the gdt/idt may not exist where the kernel thinks it is.

Make sure to initialise your virtual memory manager before initialising your gdt/idt.

### Wrapping it up
That's it for this post!
I hope this helped you fix the problem or avoid a headache with the seemingly broken vmm.

[QEMU-LOG-BLOG-POST]: https://v01d-null.github.io/understanding-qemu-logs
[RUST-PAGING]: https://os.phil-opp.com/paging-introduction/