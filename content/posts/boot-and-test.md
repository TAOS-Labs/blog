+++
title = 'Boot and Test'
date = 2025-04-19T17:02:28-05:00
draft = true
+++

## And We're Live!

I remember how this madness began. I had just finished speaking with Dr. Gheith and learned that we would receive no starter code. None. Zero. Nil. Naturally, my first question was, how do we boot?

I had previously taken [CS439 with Dr. Norman](https://www.cs.utexas.edu/~ans/classes/cs439/), and while we did briefly - for a whole of five minutes - discuss how operating systems begin their lifecycle, it was by far not enough knowledge to go out and write ourselves a bootloader from scratch.

The [OSDev Wiki](https://wiki.osdev.org/Bootloader) further convinced me: booting is surprisingly difficult and error-prone! Fortunately, the above page offered me a quick way out: [GRUB](https://wiki.osdev.org/GRUB). It's what virtually all quality kernels support, and it would certainly be enough for our needs.

Except if you look at the above page, GRUB has way too many options! Yes - it does support chainloading 10 random different operating systems, but that is not what we wanted.

I was resigned to carving out a chunk of my Thanksgiving break to try and find a good resource on GRUB and its philosophy when an acquaintance of mine told me about [Limine](https://wiki.osdev.org/Limine) and also linked me to the [Bare Bones](https://wiki.osdev.org/Limine_Bare_Bones) tutorial for getting started. At the outset, it already looked very convenient:

```c
#include <stdint.h>
#include <stddef.h>
#include <stdbool.h>
#include <limine.h>

// Set the base revision to 3, this is recommended as this is the latest
// base revision described by the Limine boot protocol specification.
// See specification for further info.

__attribute__((used, section(".limine_requests")))
static volatile LIMINE_BASE_REVISION(3);

// The Limine requests can be placed anywhere, but it is important that
// the compiler does not optimise them away, so, usually, they should
// be made volatile or equivalent, _and_ they should be accessed at least
// once or marked as used with the "used" attribute as done here.

__attribute__((used, section(".limine_requests")))
static volatile struct limine_framebuffer_request framebuffer_request = {
    .id = LIMINE_FRAMEBUFFER_REQUEST,
    .revision = 0
};

// Finally, define the start and end markers for the Limine requests.
// These can also be moved anywhere, to any .c file, as seen fit.

__attribute__((used, section(".limine_requests_start")))
static volatile LIMINE_REQUESTS_START_MARKER;

__attribute__((used, section(".limine_requests_end")))
static volatile LIMINE_REQUESTS_END_MARKER;

// GCC and Clang-mandidated functions have been omitted for brevity

// Halt and catch fire function.
static void hcf(void) {
    for (;;) {
        asm ("hlt");
    }
}

// The following will be our kernel's entry point.
// If renaming kmain() to something else, make sure to change the
// linker script accordingly.
void kmain(void) {
    // Ensure the bootloader actually understands our base revision (see spec).
    if (LIMINE_BASE_REVISION_SUPPORTED == false) {
        hcf();
    }

    // Ensure we got a framebuffer.
    if (framebuffer_request.response == NULL
     || framebuffer_request.response->framebuffer_count < 1) {
        hcf();
    }

    // Fetch the first framebuffer.
    struct limine_framebuffer *framebuffer = framebuffer_request.response->framebuffers[0];

    // Note: we assume the framebuffer model is RGB with 32-bit pixels.
    for (size_t i = 0; i < 100; i++) {
        volatile uint32_t *fb_ptr = framebuffer->address;
        fb_ptr[i * (framebuffer->pitch / 4) + i] = 0xffffff;
    }

    // We're done, just hang...
    hcf();
}
```

Contrast this to GRUB, which looks something like this:

```c
/* multiboot2.h */
#include <stdint.h>

#define MULTIBOOT2_HEADER_MAGIC 0xe85250d6
#define MULTIBOOT_ARCHITECTURE_I386 0
#define MULTIBOOT_HEADER_TAG_END 0
#define MULTIBOOT_HEADER_TAG_FRAMEBUFFER 5

struct multiboot_header {
    uint32_t magic;
    uint32_t architecture;
    uint32_t header_length;
    uint32_t checksum;
};

struct multiboot_header_tag {
    uint16_t type;
    uint16_t flags;
    uint32_t size;
};

struct multiboot_header_tag_framebuffer {
    struct multiboot_header_tag tag;
    uint32_t width;
    uint32_t height;
    uint32_t depth;
};

struct multiboot_tag {
    uint32_t type;
    uint32_t size;
};

struct multiboot_tag_framebuffer {
    struct multiboot_tag tag;
    uint64_t framebuffer_addr;
    uint32_t framebuffer_pitch;
    uint32_t framebuffer_width;
    uint32_t framebuffer_height;
    uint8_t framebuffer_bpp;
    uint8_t framebuffer_type;
    uint16_t reserved;
    // Color info omitted for brevity
};

/* kernel.c */
#include <stdint.h>
#include <stddef.h>
#include <stdbool.h>
#include "multiboot2.h"

// Multiboot2 header - using a packed struct to ensure contiguous layout
struct __attribute__((packed)) multiboot_header_complete {
    struct multiboot_header header;
    struct multiboot_header_tag_framebuffer framebuffer_request;
    struct multiboot_header_tag end_tag;
};

// Define the multiboot header with proper checksum calculation
__attribute__((section(".multiboot")))
static struct multiboot_header_complete mb_header = {
    .header = {
        .magic = MULTIBOOT2_HEADER_MAGIC,
        .architecture = MULTIBOOT_ARCHITECTURE_I386,
        .header_length = sizeof(struct multiboot_header_complete),
        // Calculate checksum to make the sum of the first four 32-bit fields zero
        .checksum = -(MULTIBOOT2_HEADER_MAGIC + MULTIBOOT_ARCHITECTURE_I386 +
                     sizeof(struct multiboot_header_complete))
    },
    .framebuffer_request = {
        .tag = {
            .type = MULTIBOOT_HEADER_TAG_FRAMEBUFFER,
            .flags = 0,
            .size = sizeof(struct multiboot_header_tag_framebuffer)
        },
        .width = 0,  // Any width is acceptable
        .height = 0, // Any height is acceptable
        .depth = 32  // Preferred depth
    },
    .end_tag = {
        .type = MULTIBOOT_HEADER_TAG_END,
        .flags = 0,
        .size = sizeof(struct multiboot_header_tag)
    }
};

// Halt and catch fire function
static void hcf(void) {
    for (;;) {
        asm volatile("hlt");
    }
}

// Frame buffer types defined by multiboot2
#define MULTIBOOT_FRAMEBUFFER_TYPE_INDEXED 0
#define MULTIBOOT_FRAMEBUFFER_TYPE_RGB     1
#define MULTIBOOT_FRAMEBUFFER_TYPE_EGA_TEXT 2

void kmain(uint32_t magic, uint32_t addr) {
    // Check if we got a valid multiboot2 response
    if (magic != 0x36d76289) {
        hcf();
    }

    // Parse multiboot2 information
    struct multiboot_tag *tag;
    struct multiboot_tag_framebuffer *fb_info = NULL;

    // Align the address to 8 bytes
    addr = (addr + 7) & ~7;

    // Skip the size field
    tag = (struct multiboot_tag *)(addr + 8);

    // Find the framebuffer tag
    while (tag->type != 0) {
        if (tag->type == 8) { // Type 8 is framebuffer
            fb_info = (struct multiboot_tag_framebuffer *)tag;
            break;
        }

        // Move to the next tag (aligned to 8 bytes)
        tag = (struct multiboot_tag *)((uint8_t *)tag + ((tag->size + 7) & ~7));
    }

    // Ensure we got a framebuffer
    if (fb_info == NULL) {
        hcf();
    }

    // Check if we have a direct RGB framebuffer
    if (fb_info->framebuffer_type != MULTIBOOT_FRAMEBUFFER_TYPE_RGB) {
        // Handle other framebuffer types or fail gracefully
        hcf();
    }

    // Draw a diagonal line
    for (size_t i = 0; i < 100 && i < fb_info->framebuffer_width &&
                       i < fb_info->framebuffer_height; i++) {
        volatile uint32_t *fb_ptr = (uint32_t *)(uintptr_t)fb_info->framebuffer_addr;
        fb_ptr[i * (fb_info->framebuffer_pitch / 4) + i] = 0xffffff;
    }

    // We're done, just hang...
    hcf();
}
```

The Limine [specification](https://github.com/limine-bootloader/limine/blob/trunk/PROTOCOL.md), by the way, offers way more out-of-the-box than the above GRUB code. For instance, it makes it extremely simple to boot up other cores, request certain kernel stack/heap size, and so on. It also takes care of switching to long mode, enabling support for paging, all of which we would have to do in GRUB ourselves.

### What about Rust?

We initially went with Zig, but due to some unforeseen circumstances - mainly the lack of stable async await infrastructure - quickly pivoted to Rust. Fortunately, it turned out that Limine already had [an established option](https://crates.io/crates/limine) for booting Rust kernels, so conversion was not too bad.

```rust
#![no_std]
#![no_main]

use limine::request::{FramebufferRequest, RequestsEndMarker, RequestsStartMarker};
use limine::BaseRevision;
use x86_64::instructions::hlt;

#[used]
#[link_section = ".requests"]
static BASE_REVISION: BaseRevision = BaseRevision::new();

#[used]
#[link_section = ".requests"]
static FRAMEBUFFER_REQUEST: FramebufferRequest = FramebufferRequest::new();

#[used]
#[link_section = ".requests_start_marker"]
static _START_MARKER: RequestsStartMarker = RequestsStartMarker::new();

#[used]
#[link_section = ".requests_end_marker"]
static _END_MARKER: RequestsEndMarker = RequestsEndMarker::new();

#[no_mangle]
extern "C" fn kmain() -> ! {
    assert!(BASE_REVISION.is_supported());

    if let Some(framebuffer_response) = FRAMEBUFFER_REQUEST.get_response() {
        if let Some(framebuffer) = framebuffer_response.framebuffers().next() {
            for i in 0..100_u64 {
                let pixel_offset = i * framebuffer.pitch() + i * 4;
                unsafe {
                    *(framebuffer.addr().add(pixel_offset as usize) as *mut u32) = 0xFFFFFFFF;
                }
            }
        }
    }

    hcf();
}

#[panic_handler]
fn rust_panic(_info: &core::panic::PanicInfo) -> ! {
    hcf();
}

fn hcf() -> ! {
    loop {
        hlt();
    }
}
```

At the outset, we use the [X86-64](https://crates.io/crates/x86_64) crate, although strictly speaking we do not need it and can just replace hcf with the following:

```rust
fn hcf() -> ! {
    loop {
        unsafe {
            core::arch::asm!("hlt");
        }
    }
}
```

However, the aforementioned crate turns out to be incredibly useful, as it made handling things like the [Interrupt Descriptor Table](https://wiki.osdev.org/Interrupt_Descriptor_Table), traversing the page tables, and accessing I/O ports almost laughably simple.

### Booting multiple cores
As previously mentioned, Limine made waking up other cores a breeze:

```rust
#![no_std]
#![no_main]
use core::sync::atomic::{AtomicBool, AtomicU64, Ordering};
use limine::request::{FramebufferRequest, RequestsEndMarker, RequestsStartMarker, SmpRequest};
use limine::smp::Cpu;
use limine::BaseRevision;
use x86_64::instructions::{hlt, interrupts};

#[used]
#[link_section = ".requests"]
static BASE_REVISION: BaseRevision = BaseRevision::new();

#[used]
#[link_section = ".requests"]
static FRAMEBUFFER_REQUEST: FramebufferRequest = FramebufferRequest::new();

#[used]
#[link_section = ".requests"]
static SMP_REQUEST: SmpRequest = SmpRequest::new();

#[used]
#[link_section = ".requests_start_marker"]
static _START_MARKER: RequestsStartMarker = RequestsStartMarker::new();

#[used]
#[link_section = ".requests_end_marker"]
static _END_MARKER: RequestsEndMarker = RequestsEndMarker::new();

static BOOT_COMPLETE: AtomicBool = AtomicBool::new(false);
static CPU_COUNT: AtomicU64 = AtomicU64::new(0);

#[no_mangle]
extern "C" fn kmain() -> ! {
    assert!(BASE_REVISION.is_supported());


    if let Some(framebuffer_response) = FRAMEBUFFER_REQUEST.get_response() {
        if let Some(framebuffer) = framebuffer_response.framebuffers().next() {
            for i in 0..100_u64 {
                let pixel_offset = i * framebuffer.pitch() + i * 4;
                unsafe {
                    *(framebuffer.addr().add(pixel_offset as usize) as *mut u32) = 0xFFFFFFFF;
                }
            }
        }
    }

    let smp_response = SMP_REQUEST.get_response().expect("SMP request failed");
    let cpu_count = smp_response.cpus().len() as u64;


    let bsp_id = smp_response.bsp_lapic_id();
    for cpu in smp_response.cpus() {
        if cpu.id != bsp_id {
            cpu.goto_address.write(secondary_cpu_main);
        }
    }

    while CPU_COUNT.load(Ordering::SeqCst) < cpu_count - 1 {
        core::hint::spin_loop();
    }

    BOOT_COMPLETE.store(true, Ordering::SeqCst);
    hcf();
}

#[no_mangle]
unsafe extern "C" fn secondary_cpu_main(cpu: &Cpu) -> ! {
    CPU_COUNT.fetch_add(1, Ordering::SeqCst);

    while !BOOT_COMPLETE.load(Ordering::SeqCst) {
        core::hint::spin_loop();
    }

    hcf();
}

#[panic_handler]
fn rust_panic(info: &core::panic::PanicInfo) -> ! {
    hcf();
}

fn hcf() -> ! {
    loop {
        interrupts::disable();
        hlt();
    }
}
```

It's, for the most part, pretty straightforward:
- We ask Limine for the information about available CPUs using the SMP request
- For each CPU that isn't the BSP, we set its entry point to our secondary_cpu_main function
- We use atomic variables to coordinate between cores:
    - CPU\_COUNT tracks how many secondary cores have started up
    - BOOT\_COMPLETE signals when it's safe for secondary cores to proceed
- The main core waits in a spin loop until all secondary cores have checked in
- Secondary cores increment the counter and wait for the "all clear" signal
- Once everyone's ready, we set BOOT_COMPLETE and all cores halt

Apart from it not being terribly exciting, we require no complex synchronization primitives besides atomics.

