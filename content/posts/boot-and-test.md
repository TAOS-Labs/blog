+++
title = 'Boot and Test'
date = 2025-04-19T17:02:28-05:00
draft = false
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

### A note on Limine's protocol
As you have just seen, Limine works via requests and responses. This protocol allows the operating system kernel to communicate with the bootloader by setting up specific memory structures that Limine recognizes. The kernel makes requests for various system resources or information by populating designated memory areas with specific tags and parameters, and Limine responds by filling in the appropriate data structures with the requested information, such as memory maps, module locations, or hardware capabilities. This request-response architecture creates a clean interface between the bootloader and kernel, providing flexibility while maintaining compatibility across different system configurations.

### What about Rust?
We initially went with Zig, but due to some unforeseen circumstances - mainly the lack of stable [async await infrastructure](https://github.com/ziglang/zig/issues/18873) - quickly pivoted to Rust. Fortunately, it turned out that Limine already had [an established option](https://crates.io/crates/limine) for booting Rust kernels, so conversion was not too bad. It looks something like this:

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

-   We ask Limine for the information about available CPUs using the SMP request
-   For each CPU that isn't the BSP, we set its entry point to our secondary_cpu_main function
-   We use atomic variables to coordinate between cores:
    -   CPU_COUNT tracks how many secondary cores have started up
    -   BOOT_COMPLETE signals when it's safe for secondary cores to proceed
-   The main core waits in a spin loop until all secondary cores have checked in
-   Secondary cores increment the counter and wait for the "all clear" signal
-   Once everyone's ready, we set BOOT_COMPLETE and all cores halt

Apart from the end result not being terribly exciting, we require no complex synchronization primitives besides atomics.

That `bsp_lapic_id`, by the way, is the Local Advanced Programmable Interrupt Controller [(LAPIC)](https://wiki.osdev.org/APIC) ID for the Bootstrap Processor — essentially a unique identifier for the primary core that's handling the boot process...

### What about tests?

We quickly discovered that testing kernels in Rust was not so much fun due to the inability to use the built-in testing framework. Fortunately, Rust being Rust presented us with an alternative: the [custom test frameworks](https://doc.rust-lang.org/beta/unstable-book/language-features/custom-test-frameworks.html) seemed to be specifically designed for this purpose by allowing one to redefine what testcases and test runners looked like on bare metal. TAOS' setup for this is pretty straightforward:

```rust
// in main.rs:
// Enable the unstable custom_test_frameworks feature to create our own test runner
#![feature(custom_test_frameworks)]
// Specify our custom test_runner function defined in the taos module
#![test_runner(taos::test_runner)]
// Generate a test_main function that will be called to run tests
#![reexport_test_harness_main = "test_main"]

// Somewhere, in your entry function:
#[no_mangle]
extern "C" fn _start() -> ! {
    // Do init if necessary - this would initialize hardware, memory, etc

    // Only compile this section when running tests
    #[cfg(test)]
    test_main();   // Call the generated test runner function

    // Normal code path
}

/// Production panic handler.
#[cfg(not(test))]
#[panic_handler]
fn rust_panic(info: &core::panic::PanicInfo) -> ! {
    // Production-level, I.e, any run other than tests, panics go here
}

/// Test panic handler.
#[cfg(test)]
#[panic_handler]
fn rust_panic(info: &core::panic::PanicInfo) -> ! {
    // Test panics go here
}
```

The rest of our setup was also relatively straightforward and was at the library root:

```rust
// In lib.rs
// When testing, mark this as no_main since we'll have our own entry point
#![cfg_attr(test, no_main)]
// Enable custom test frameworks feature here too
#![feature(custom_test_frameworks)]
// Define our test runner within this crate
#![test_runner(crate::test_runner)]
// Export the test_main function that will run our tests
#![reexport_test_harness_main = "test_main"]

// Define a type alias for futures to make code more readable
// Pin<Box<...>> is needed because async functions return futures that may contain self-references
// which must be pinned in memory to be safe
type BoxFuture<T> = Pin<Box<dyn Future<Output = T> + Send + 'static>>;

pub trait Testable: Sync {
        // Each test returns a future that completes when the test is done
    fn run(&self) -> BoxFuture<()>;
    // Return the name of the test for reporting
    fn name(&self) -> &str;
}

// Implement Testable for any function that returns a future with () output
// This allows us to write async test functions
impl<F, Fut> Testable for F
where
    F: Fn() -> Fut + Send + Sync + 'static,  // F is any function that can be called multiple times
    Fut: Future<Output = ()> + Send + 'static,  // The function returns a future that resolves to ()
{
    fn run(&self) -> BoxFuture<()> {
        // Call the function and box the resulting future
        Box::pin(self())
    }

    fn name(&self) -> &str {
        // Use the type name of the function as the test name
        core::any::type_name::<F>()
    }
}

pub fn test_runner(tests: &[&(dyn Testable + Send + Sync)]) {
    println!("Running {} tests\n", tests.len());

    // Collect all test futures with their names for execution
    let test_futures: alloc::vec::Vec<_> = tests
        .iter()
        .map(|test| {
            let name = alloc::string::String::from(test.name());
            (name, test.run())
        })
        .collect();

    // Create a future that runs all tests sequentially and reports results
    let future = async move {
        for (name, fut) in test_futures {
            print!("test {}: ", name);
            fut.await; // Wait for the test to complete
            println!("ok");
        }
        println!("\ntest result: ok.");
        exit_qemu(QemuExitCode::Success);
    };

    // Schedule the future to run on kernel
}

// Special panic handler for tests that reports failures
pub fn test_panic_handler(info: &core::panic::PanicInfo) -> ! {
    println!("FAILED");
    println!("Error: {}\n", info);
    exit_qemu(QemuExitCode::Failed);
    idle_loop();   // Should never be reached but needed for the ! return type
}

// Define exit codes for QEMU that indicate test status
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u32)]
pub enum QemuExitCode {
    Success = 0x10,
    Failed = 0x11,
}

// Function to exit QEMU emulator with a status code
// This allows test results to be communicated back to the host OS
pub fn exit_qemu(exit_code: QemuExitCode) {
    use x86_64::instructions::port::Port;

    unsafe {
        // Write to a special I/O port that the QEMU emulator monitors
        let mut port = Port::new(0xf4);
        port.write(exit_code as u32);
    }
}

// Special entry point for running tests in the library itself
#[cfg(test)]
#[no_mangle]
pub extern "C" fn _start() -> ! {
    println!("!!! RUNNING LIBRARY TESTS !!!");
    // Do init
    test_main();
    unsafe {
        // Run loop
        // FOr us tests were ran on bsp
    };
}

// Panic handler specifically for tests in the library
#[cfg(test)]
#[panic_handler]
fn panic(info: &core::panic::PanicInfo) -> ! {
    test_panic_handler(info)   // Delegate to our custom test panic handler
}
```

We require 2 entry points because Rust's test framework needs separate execution contexts for kernel-level tests and library-level tests. The _start function in main.rs serves as the entry point for the entire operating system during normal execution and when testing kernel functionality in its entirety. Meanwhile, the _start function in lib.rs - only compiled during test runs - creates an isolated test environment specifically for running unit tests on library components independently from the full kernel. This dual-entry approach allows developers to test both integrated system behavior through the main entry point and isolated library components through the library entry point, all while using the same custom test framework but with appropriate scoping for what's being tested.

One could also in theory support both async and standard tests, with the latter being transformed to the former via something like [Procedural Macros](https://doc.rust-lang.org/reference/procedural-macros.html). This would involve defining a new macro, traversing the token streams for whatever has been decorated with it (the proc-macros crate provides utilities to do this), and, as a naive starting point, check that the function(s) have not been marked as async. We actually did get this to work, but did not find it to be terribly useful. Most of our system ended up using async/await patterns, and the additional complexity of procedural macros for the one percent of tests was deemed to not be worthwhile to keep around.

### So how do I run this thing?
Tests are great... assuming one could actually run them. Unfortunately, here is where our luck ran out. From brief research into OS development in Rust, a lot of people seem to favor the approach shown in [Philipp Oppermann's blog](https://os.phil-opp.com/). That is, they use the [bootimage](https://crates.io/crates/bootimage) crate, which in turn depends on [bootloader](https://crates.io/crates/bootloader).

Since we chose Limine, the above resources were not an option for us. Fortunately, someone created a derivative of bootimage called, you guessed it, [limage](https://github.com/phillipg14/limage). Unfortunately, said tool was rather limited in what it did and was somewhat focused on the maintainer's hobby OS. We repurposed it, keeping the name and giving the crate a [new home](https://github.com/TAOS-Labs/limage/). The readme is mostly up-to-date, although we did develop a separate file designed to configure specifically the crate itself and the arguments it passes to qemu. It looks something like this:

```toml
[build]
image_path = "target/kernel.iso"

[qemu]
binary = "qemu-system-x86_64"
base_args = [
    # Memory
    "-m", "2G",

    # SMP settings
    "-smp", "2",
    "-cpu", "Conroe-v1,+x2apic,+invtsc",

    # USB
    "-device", "qemu-xhci",

    # Network
    "-netdev", "user,id=net0",
    "-device", "usb-net,netdev=net0",

    # Audio
    "-device", "intel-hda",
    "-device", "hda-duplex",

    # General Storage
    "-drive", "id=mysdcard,file=storage_test.img,if=none,format=raw",
    "-device", "sdhci-pci",
    "-device", "sd-card,drive=mysdcard",

    # Graphics
    "-vga", "std",

    # Debugging traces
    # USB
    # "-trace", "usb*",
    # SD card
    # "-trace", "sd*",

    # Boot media and UEFI settings
    "-cdrom", "{image}",
    "-drive", "if=pflash,unit=0,format=raw,file={ovmf}/ovmf-code-x86_64.fd,readonly=on",
    "-drive", "if=pflash,unit=1,format=raw,file={ovmf}/ovmf-vars-x86_64.fd"
]

[modes]
terminal = { args = ["-nographic"] }
gui = { args = [] }
gdb-terminal = { args = ["-nographic", "-s", "-S"] }
gdb-gui = { args = ["-s", "-S"] }

[test]
timeout_secs = 30
success_exit_code = 33
no_reboot = true
extra_args = [
    "-device", "isa-debug-exit,iobase=0xf4,iosize=0x04",
    "-serial", "stdio",
    "-display", "none"
]
```

This gave us the ability to define different modes our system could run in. For instance, we could type "cargo run mode terminal" and disable the graphics display, or we could ask qemu to run gdb in gui mode by typing "cargo run mode gdb-gui".

Unfortunately, due to the lack of time, limage was a bit limited in what it could do. For instance, it does not currently support conditional platform flags for something like Mac versus Linux. Had we had more time, more effort would have been spent to try and repurpose the cargo run command to work with existing infrastructure like Make. Failing this, we would have added the ability to indicate whether a particular argument defined in the Limage configuration file was for Mac versus Linux.

### Automating tests
We have had the misfortune of merging code into main that broke tests, all because someone did not bother to verify they were still passing. Surprisingly, few resources exist online for how to make Github actions test the OS as part of a pull request. After some tinkering, we came up with the following workflow:

```yaml
name: Rust

on:
    push:
        branches: ["main"]
    pull_request:
        branches: ["main"]
    workflow_dispatch:
        inputs:
            qemu_version:
                description: "QEMU version to install"
                required: false
                default: "9.2.0"
                type: string

env:
    CARGO_TERM_COLOR: always
    QEMU_VERSION: ${{ github.event.inputs.qemu_version || '9.2.0' }}

jobs:
    lint:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v4

            - name: Setup toolchain
              uses: actions-rust-lang/setup-rust-toolchain@v1
              with:
                  components: rustfmt, clippy
                  toolchain: nightly-x86_64-unknown-linux-gnu
                  rustflags: ""

            - name: Add rust-src to toolchain
              run: rustup component add rust-src --toolchain nightly-x86_64-unknown-linux-gnu

            - name: Add x86_64 target
              run: rustup target add x86_64-unknown-none

            - uses: Swatinem/rust-cache@v2
              with:
                  workspaces: "kernel -> target"

            - name: Run linting checks
              run: make check

    test:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v4

            - name: Install system dependencies
              run: |
                  sudo apt-get update
                  sudo apt-get install -y xorriso e2fsprogs build-essential ninja-build pkg-config libglib2.0-dev libpixman-1-dev libslirp-dev

            - name: Build and install QEMU
              run: |
                  echo "Building QEMU ${QEMU_VERSION}"
                  sudo apt-get install -y python3 python3-pip git libcap-ng-dev libattr1-dev libzstd-dev
                  # Install Python dependencies needed by QEMU
                  sudo pip3 install tomli sphinx sphinx_rtd_theme

                  wget https://download.qemu.org/qemu-${QEMU_VERSION}.tar.xz
                  tar xvf qemu-${QEMU_VERSION}.tar.xz
                  cd qemu-${QEMU_VERSION}
                  ./configure --target-list=x86_64-softmmu --enable-slirp --enable-curses --enable-tools
                  make -j$(nproc)
                  sudo make install
                  qemu-system-x86_64 --version

            - name: Setup toolchain
              uses: actions-rust-lang/setup-rust-toolchain@v1
              with:
                  components: rustfmt, clippy
                  toolchain: nightly-x86_64-unknown-linux-gnu
                  rustflags: ""

            - name: Add rust-src to toolchain
              run: rustup component add rust-src --toolchain nightly-x86_64-unknown-linux-gnu

            - name: Add x86_64 target
              run: rustup target add x86_64-unknown-none

            - name: Install limage
              run: cargo install --git https://github.com/TAOS-Labs/limage

            - uses: Swatinem/rust-cache@v2
              with:
                  workspaces: "kernel -> target"

            - name: Create blank drive
              run: make blank_drive

            - name: Run tests
              run: make test
```

The workflow consists of two jobs: lint and test. The lint job is responsible for code quality checks without actually running any code. It sets up a nightly Rust toolchain with formatting and static analysis tools like clippy. This ensures our code adheres to Rust's idioms and catches common mistakes before they make it to testing. The job runs on Ubuntu and caches dependencies to speed up subsequent runs.

The test job is more interesting and tackles the challenge of testing an operating system in CI. It provisions an Ubuntu runner and installs all the necessary dependencies for building QEMU from source. We opted for building QEMU directly rather than using package manager versions because at the time of writing, Qemu version 9.2, which is what we use, does not exist on managers like apt, or at least that was the case with Ubuntu 24. The job then sets up the same Rust toolchain as the lint job, installs limage, creates a blank drive for the OS to write to, and finally runs the tests. The tests boot our OS in QEMU and verify functionality in a real environment, not just through unit tests.

It should be noted that we currently do not attempt to cache the Qemu emulator and rebuild it from scratch on every pull request or push to main. Doing so has proven to be surprisingly difficult and we decided that it was simply not worth the trouble to figure out.

