+++
title = 'Virtual Memory'
date = 2025-04-19T17:02:28-05:00
draft = false
author = 'Kshitij Kapoor'
+++

Written by: Kshitij Kapoor
## Accessing Memory to Access Memory

The most interesting description of a computer that I have ever heard was the following: "A computer is state held in memory, and various external components are modifying that state".

This description, of course, came from Dr. Gheith, in the second lecture of his Multicore Operating Systems course. So, given the importance that that description places on memory - and, by extension, virtual memory - and the fact that I was starting work on the virtual memory system for TAOS just a few days later, it's needless to say that I was a little overwhelmed. 

The following write-up will serve two purposes. One, it will be a loose guide on how virtual memory works and how one can go about implementing it. Two, it will be a cautionary tale of my implementation and mistakes that I made along the way. Whatever you want to read it as, I hope that it is at least interesting. 

### A start - what does the kernel see?

In an introductory operating systems course, you may learn that processes' address spaces deal entirely with virtual addresses. Taking this idea to its logical conclusion raises the question - how does the kernel hand out physical memory and create those mappings? It would make sense that it, too, deals only in virtual memory - or, can you just break the idea of a page table when you have privilege?

I had all of these questions when I started this project, and the real methodology here is quite interesting. 

The ultimate conclusion here is that, at least for x86, the kernel must be mapped. There is simple reason for this - every single instruction fetch or data access **must** go through the page-translation hardware. There is no way to avoid this, it is literally baked into the architecture. 

So, what can we do about this?

There are some neat consequences of a reasonable virtual memory system that allow for an elegant setup here:
- First, virtual addresses are plentiful - having 2^64 (or, really, 2^48 if you consider [canonical addressing](https://stackoverflow.com/questions/25852367/x86-64-canonical-address)) means that if you are running out of virtual addresses, a traditional OS is not truly suitable for your workload.
- Second, privilege level restrictions on memory can be enforced by the hardware - there are bits on page table entries that can enforce this.

So, knowing this - we can create an extremely nice partition, where the kernel lives in the "higher half" of virtual memory, and user space works solely in the lower half. This, in turn, comes with many benefits:
1. Strong, hardware supported isolation
2. Fast privilege level switch (no need to load a new page table when running privileged code like a system call)
3. Still allows for large virtual addresses for users

#### How do we actually accomplish this?

For us, this was a simple task. For booting, we used [Limine](https://wiki.osdev.org/Limine), a modern alternative to GRUB.  Limine has a lot of niceties: it sets up some page table structures, boots into long mode, and, most importantly, loads the kernel into the higher half of memory and sets up an important component - "Higher Half Direct Mapping".

### Higher Half Direct Mapping

Previously, we alluded to the idea that we would need some way to examine physical memory in the kernel. Higher Half Direct Mapping, or HHDM, helps us accomplish this by reserving a contiguous "window" of virtual addresses (starting at what is known as "HHDM Offset" or "HHDM Base") that directly map to physical memory. 

So, we can get something like:
```
virtual =  hhdm_base + p
physical = virtual − hhdm_base
```

So, effectively, our memory system initialization looks like the following: 
1. The BSP will activate the paging system (technically, it is already activated, but we create software data structures to interface with it here)
2. We will create a simple bootstrapping frame allocator for now - we will discuss the reason why this is necessary later (and why it might not be!)
3. We will enable the `NO_EXECUTE EFER` flag which allows us to mark pages as unexecutable for aforementioned security reasons
4. We will initialize the Kernel Heap

In code, this looks like the following:
```rust
lazy_static! {
    // The kernel mapper
    pub static ref KERNEL_MAPPER: Mutex<OffsetPageTable<'static>> = Mutex::new(unsafe { paging::init() });
    // Start of kernel virtual memory
    pub static ref HHDM_OFFSET: VirtAddr = VirtAddr::new(
        HHDM_REQUEST
            .get_response()
            .expect("HHDM request failed")
            .offset()
    );
}

/// Initializes the global frame allocator and kernel heap
///
/// * `cpu_id`: The CPU to initialize for. We only want to initialize a frame allocator for cpuid 0
pub fn init(cpu_id: u32) {
    if cpu_id == 0 {
        unsafe {
            *FRAME_ALLOCATOR.lock() =
                Some(GlobalFrameAllocator::Boot(BootIntoFrameAllocator::init()));
        }

        unsafe {
            // Must be done after enabling long mode + paging
            // Allows us to mark pages as unexecutable for security
            Efer::update(|flags| {
                flags.insert(EferFlags::NO_EXECUTE_ENABLE);
            });
        }
        heap::init_heap().expect("Failed to initialize heap");
    }
}
```

A key benefit of HHDM is that now the kernel has a "picture" of physical memory. From this, we can make reasonable page table data structures.`
### Bootstrapping frame allocator

Our design invokes the use of a "bootstrapping" frame allocator - something to just get us going. This is likely not _actually_ necessary - you can do some math to figure most of the following information out. However, we chose to do it in this manner for convenience.

So, why did we in our design need this?

The answer is simple: We needed a kernel heap, and, because the kernel is mapped, we need some frames to hand out for the kernel heap. So, we create a Bootstrapping frame allocator with the simple purpose of being able to hand out frames to our kernel heap, which in turn lets us initialize our heap. Limine is nice again here - it tells us how what portions of physical memory we can actually hand out.

```rust
    /// Function that gives an iterator over sections of usable memory
    ///
    /// # Returns
    /// Returns an iterator of usable PhysFrames
    fn usable_frames(&self) -> impl Iterator<Item = PhysFrame> + '_ {
        self.memory_map
            .entries()
            .iter()
            .filter(|r| r.entry_type == EntryType::USABLE)
            .flat_map(|r| (r.base..(r.base + r.length)).step_by(4096))
            .filter(move |&addr| {
                addr < self.kernel_start
                    || addr >= self.kernel_end
                    || addr < HEAP_START as u64
                    || addr > (HEAP_START as u64).wrapping_add(HEAP_SIZE as u64)
            })
            .map(|addr| PhysFrame::containing_address(PhysAddr::new(addr)))
    }

    /// Give all frames allocated so far
    ///
    /// # Returns
    /// Returns an iterator over all frames allocated so far
    pub fn allocated_frames(&self) -> impl Iterator<Item = PhysFrame> + '_ {
        let start_addr = self
            .first_frame
            .expect("No frames allocated yet")
            .start_address()
            .as_u64();

        (0..self.allocated_count).map(move |i| {
            let addr = start_addr + (i as u64 * FRAME_SIZE as u64);
            PhysFrame::containing_address(PhysAddr::new(addr))
        })
    }
}

```

As you can see, the code above is incredibly simple - just iterate over the usable frames and hand out the next possible one.  This let's us set up our heap nicely, for which we used [talc](https://github.com/SFBdragon/talc). For brevity, I have omitted the discussion of the specifics of this heap.	

### A smarter frame allocator

Obviously, the frame allocator we have right now is not actually very serviceable. It realistically cannot even free frames, at least in any reasonable manner. Thus, it is only for bootstrapping. We can use the frames we have allocated to the heap (with those virtual to physical mappings made) to create the data structures necessary to have our real physical memory management. For this, we use a buddy frame allocator.

### Buddy frame allocator

A buddy allocator is a surprisingly elegant solution to a very un-elegant problem: how do you manage thousands of fixed-size memory frames in a way that doesn't turn into chaos the moment someone asks for memory of a different size, or dares to free it? And, how do you still keep efficiency?

The idea behind this allocator is simple: `2^n * PAGE_SIZE` sized blocks of contiguous physical memory, and we keep track of a free list for each order `n`.

From this, if we want a block of size `2^k * PAGE_SIZE`, and the smallest available block if of a larger size, we can just split it again and again until we get a correctly sized block. This leads to `O(log N)`allocation of frames.

Freeing a block works similarly - just check if a "buddy" is also free right now, and then merge those blocks.

None of this is super important - a simple bitmap would have been fine for this project (and in fact, that's what we started with). It is, however, interesting.

For completeness, here is our implementation of an allocation. It works very well!

```rust
/// Allocates a single frame (Order 0 of Allocator)
/// If order 0 block does not exist yet, splits a higher
/// order block
/// Runs in O(Log N) time
///
/// # Returns
/// Returns either a PhysFrame or None if not available
fn allocate_frame(&mut self) -> Option<PhysFrame> {
	let mut found_order = None;

	// we want order 0, so find the closest to that
	for order in 0..=self.max_order {
		if !self.free_lists[order].is_empty() {
			found_order = Some(order);
			break;
		}
	}

	let mut order = found_order? as u16;

	let block_index = self.free_lists[order as usize]
		.pop()
		.expect("Expected something to be in list");

	// if the block we found is greater than order 0, split
	while order > 0 {
		order -= 1;

		let buddy = block_index + (1 << order);

		self.frames[buddy].order.store(order, Ordering::Relaxed);
		self.free_lists[order as usize].push(buddy);
	}

	self.allocated_count += 1;
	self.free_count -= 1;

	// update frame descriptor
	let desc = &self.frames[block_index];
	desc.ref_count.fetch_add(1, Ordering::Relaxed);
	let addr = block_index * PAGE_SIZE;

	Some(PhysFrame::containing_address(PhysAddr::new(addr as u64)))
```


### Virtual Memory

Finally, after all of that background, we can start talking about virtual memory. As mentioned before, we used the x86_64 crate, which provides many data structures and functions for paging. An example of our utilization can be found here:
```rust
/// Creates a mapping
/// Default flags: PRESENT | WRITABLE
///
/// # Arguments
/// * `page` - a Page that we want to map
/// * `mapper` - anything that implements a the Mapper trait
/// * `flags` - Optional flags, can be None
///
/// # Returns
/// Returns the frame that was allocated and mapped to this page
pub fn create_mapping(
    page: Page,
    mapper: &mut impl Mapper<Size4KiB>,
    flags: Option<PageTableFlags>,
) -> PhysFrame {
    let frame = alloc_frame().expect("no more frames");

    let _ = unsafe {
        mapper
            .map_to(
                page,
                frame,
                flags.unwrap_or(
                    PageTableFlags::PRESENT
                        | PageTableFlags::WRITABLE
                        | PageTableFlags::USER_ACCESSIBLE,
                ),
                FRAME_ALLOCATOR
                    .lock()
                    .as_mut()
                    .expect("Global allocator not initialized"),
            )
            .expect("Mapping failed")
    };

    frame
}

```

This is a simple function that allocates a new frame and creates a mapping to that frame, and is used widely for virtual memory management. We take in a generic mapper (allowing us to use either the kernel page table or user page tables), and return the frame that we allocate.  The map_to() function updates page tables and allocates new frames if necessary (for instance, if an intermediate page table did not already exist).

You'll notice an alloc_frame() function here - this is a function that uses whatever current allocator we have (Boot for initialization, Buddy for everything after) and updates those metadata structures as needed. It's convenient that this generic function exists, as it allows for much cleaner code.

### An important problem: TLB Shootdowns

An important aspect of virtual memory is the TLB, which is a per-core cache for page to frame translations (this massively speeds up the virtual memory system under the concepts of temporal and spatial locality). 

Let's say you have a process running on CPU 0. It has a mapped page at some virtual address, let's say `0xdeadbeef`. Let's say this maps to frame 10. Now, let's say that, for whatever reason, something about this page table entry changes. Maybe it was unmapped, maybe permissions changed. Something happens. For simplicity's sake, let's say an unmap happens.

Now, CPU 0 tries to access this page, which happened to be cached in the core's TLB before. CPU 0 would simply go through the TLB - this means that a process would be able to access an unmapped frame - which is not only nondeterministic behavior, it is also likely a massive security vulnerability.  

This problem only gets worse when we realize that multiple cores exist, and they do not share TLBs. This means that when something of this nature happens, all cores must invalidate that entry in their TLBs.

So, how do we deal with this?

For us, we decided to use Inter-processor-interrupts, or IPIs, to deliver messages to other cores to invalidate a TLB. This was implemented in the following way:

```rust
//! Translation Lookaside Buffer Shootdowns
//!
//! - Exposes a function to perform TLB Shootdowns

use crate::{
    constants::{idt::TLB_SHOOTDOWN_VECTOR, MAX_CORES},
    interrupts::x2apic::{current_core_id, send_ipi, TLB_SHOOTDOWN_ADDR},
};
use core::arch::asm;
use x86_64::VirtAddr;

/// Sends an inter-process interrupt to all other cores to clear TLB entry with a specific VA
///
/// # Arguments:
/// * target_vaddr: VA that has to be flushed in all TLBs
pub fn tlb_shootdown(target_vaddr: VirtAddr) {
    let current_core = current_core_id();
    let vaddr = target_vaddr.as_u64();

    {
        // Acquire the lock and update all cores except the current one.
        let mut addresses = TLB_SHOOTDOWN_ADDR.lock();
        for core in 0..MAX_CORES {
            if core != current_core {
                addresses[core] = vaddr;
                send_ipi(core as u32, TLB_SHOOTDOWN_VECTOR);
            }
        }
    }

    unsafe {
        asm!("invlpg [{}]", in(reg) vaddr, options(nostack, preserves_flags));
    }
}

```

The above code writes to some shared memory region (between cores) which address to invalidate next (a simple spin lock is used here - anything more would likely be too much overhead). 

Then, an IPI is delivered to each core, and these cores now must run this handler: 
```rust
// TODO Technically, this design means that when TLB Shootdows happen, each core must sequentially
// invalidate its TLB rather than doing this in parallel. While this is slow, this is of low
// priority to fix
#[no_mangle]
extern "x86-interrupt" fn tlb_shootdown_handler(_: InterruptStackFrame) {
    let core = current_core_id();
    {
        let mut addresses = TLB_SHOOTDOWN_ADDR.lock();
        let vaddr_to_invalidate = addresses[core];
        if vaddr_to_invalidate != 0 {
            unsafe {
                core::arch::asm!("invlpg [{}]", in (reg) vaddr_to_invalidate, options(nostack, preserves_flags));
            }
            addresses[core] = 0;
        }
    }
    x2apic::send_eoi();
}
```

Here, the invalid invlpg (invalidate page) instruction is used to invalidate a specific entry in the TLB for a core.

If you read the TODO comment, you may notice a funny quirk of this design - each core must sequentially invalidate the TLB, which seems ridiculous considering the reason that multiple cores exist in the first place. But, as mentioned, it was of low priority to fix.

Another quick aside: If you choose to work on a project of this nature, please be sure to send eoi's for every interrupt - forgetting to do so is a pain to debug, and I can tell you that, unfortunately, from experience.

Finally, how do we actually know that this works, and that TLB's are invalidated? Well, here is a test that proves this. The comment does a good job explaining what it does, so I'll let it speak for itself.

```rust
    /// Tests that TLB shootdowns work correctly across cores.
    ///
    /// This test creates a mapping for a given page and writes a value to cache it on the current core.
    /// It then schedules a read of that page on an alternate core (to load it into that core’s TLB cache).
    /// After that, the mapping is updated to a new frame with new contents and the new value is written.
    /// Finally, the test re-schedules a read on the alternate core and verifies that the new value is observed.
    #[test_case]
    async fn test_tlb_shootdowns_cross_core() {
        const AP: u32 = 1;
        const PRIORITY: usize = 3;

        // Create mapping and set a value on the current core.
        let page: Page = Page::containing_address(VirtAddr::new(0x500000000));

        {
            let mut mapper = KERNEL_MAPPER.lock();
            let _ = create_mapping(page, &mut *mapper, None);
            unsafe {
                page.start_address()
                    .as_mut_ptr::<u64>()
                    .write_volatile(0xdead);
            }
        }

        // Mapping exists now and is cached for the first core.

        // Schedule a read on core 1 to load the page into its TLB cache.
        schedule_kernel_on(AP, async move { pre_read(page).await }, PRIORITY);

        while PRE_READ.load(Ordering::SeqCst) == 0 {
            core::hint::spin_loop();
        }

        {
            let mut mapper = KERNEL_MAPPER.lock();

            let new_frame = alloc_frame().expect("Could not find a new frame");

            // Update the mapping so that a TLB shootdown is necessary.
            update_mapping(
                page,
                &mut *mapper,
                new_frame,
                Some(PageTableFlags::PRESENT | PageTableFlags::WRITABLE),
            );

            unsafe {
                page.start_address()
                    .as_mut_ptr::<u64>()
                    .write_volatile(0x42);
            }
        }

        // Schedule another read on core 1 to verify that the new value is visible.
        schedule_kernel_on(AP, async move { post_read(page).await }, PRIORITY);

        while POST_READ.load(Ordering::SeqCst) == 0 {
            core::hint::spin_loop();
        }

        assert_eq!(POST_READ.load(Ordering::SeqCst), 0x42);

        let mut mapper = KERNEL_MAPPER.lock();
        remove_mapped_frame(page, &mut *mapper);
    }

```


### VMAs - leaving hardware

Most of what we have discussed so far has been focused on the hardware itself. Now, we can leave that universe and start actually using virtual memory, including, in tandem with other systems, begin using it to support actual processes. Let's say a user process wants to allocate memory with `mmap` or `brk`, or one tries to map a file - what happens then? 

This is where VMAs, or Virtual Memory Areas, come in. 

A VMA is, at its core, a range of virtual memory that has a meaning. That meaning could be:
- This memory is backed by zero-initialized anonymous pages (no file exists)
- This is a read-only mapping of a file
- This is the heap
- This is the executable code

Every process has a set of VMAs — a kind of high-level map of the address space. The page tables back this up, but they don’t store metadata. The VMA is where we remember _why_ a page exists, and what to do when it doesn’t (yet - think lazy loading).

I'll leave it as an exercise to the reader to determine why this might be useful.

One quick note: some people may learn about VMAs and think that they are effectively software page tables. This assertion is incorrect. VMAs do not replace the notion of page tables. While page tables tell us _how_  to get to physical memory, VMAs describe _what_ memory might actually be. 

In TAOS, the virtual memory of a process is entirely backed by VMAs (something like the code segment would be backed by a file, whereas the stack would be anonymously backed).

Our virtual memory areas are organized into a BTreeMap, keyed by start address. We also guarantee that no VMA overlaps with one another - we have coalescing in order to prevent this. 

This is the declaration of our VMA struct:
```rust
/// A VMA now stores multiple backing segments, each for a subrange.
#[derive(Clone, Debug)]
pub struct VmArea {
    /// Startinng address for this VmArea
    pub start: u64,
    /// Ending addss for this VmArea
    pub end: u64,
    /// Mapping from an offset (relative to `start`) to a segment.
    pub segments: Arc<Mutex<BTreeMap<u64, VmAreaSegment>>>,
    /// Permission and behavior flags for this VmArea
    pub flags: VmAreaFlags,
}
```

As well as other interesting declarations:
```rust
/// A segment within a VmArea representing a subrange with its own backing.
#[derive(Clone, Debug)]
pub struct VmAreaSegment {
    /// Start offset relative to the VmArea's start.
    pub start: u64,
    /// End offset relative to the VmArea's start.
    pub end: u64,
    /// The backing for this segment.
    pub backing: Arc<VmAreaBackings>,
    /// File descriptor backed by this segment
    pub fd: usize,
    /// Offset into the file where we are
    pub pg_offset: u64,
}

/// Anonymous VM area for managing reverse mappings for anonymous pages.
#[derive(Debug)]
pub struct VmAreaBackings {
    /// VMAs keyed by offset.
    pub mappings: Mutex<BTreeMap<u64, Arc<VmaChain>>>,
    /// TODO: Used for reverse mappings
    pub vmas: Mutex<Vec<Arc<Mutex<VmArea>>>>,
}

/// Reverse mapping chain entry linking an offset to a physical page.
#[derive(Debug)]
pub struct VmaChain {
    /// Offset into the backing that contains this VmaChain
    pub offset: u64,
    /// The frame that this VmaChain maps to
    pub frame: PhysFrame,
}
```

There are a lot of data structures and BTreeMaps here, so I am sure it could get a little confusing. Here is a brief description of how they work:

`VmAreaSegment`
A segment is a subrange of a `VmArea`. Each one represents a region with a consistent backing (i.e. where the data comes from). It tracks:
- where the segment starts and ends (relative to the start of the VMA),
- the backing (`Arc<VmAreaBackings>`),
- the file descriptor (if this is file-backed),
- and the page offset into that file.
This setup allows us to split and coalesce VMAs while preserving the origin of each subregion.

`VmAreaBackings`
This is shared state for backing memory. A single `VmAreaBacking` might be used by multiple processes (e.g. due to fork or file mapping). It holds:
- a BTreeMap from page-aligned offsets to actual frames (`VmaChain`),
- and a list of all VMAs that reference it (useful for reverse mappings or when updating shared pages).

`VmaChain`
A single mapping from a page-aligned offset to a physical frame. These are what let us lazily allocate and track which pages exist and where they live physically.

So, what happens on a page fault?
1. We find the `VmArea` the faulting address belongs to.
2. Inside it, we find the corresponding `VmAreaSegment`.
3. From that segment, we get the `VmAreaBackings`, and compute the page-aligned offset within it.
4. If a `VmaChain` already exists at that offset, we use its frame. If not, we allocate a frame, zero it (or read from file), and insert a new `VmaChain`.
5. Finally, we map the faulting page to that frame and continue execution.

So, why actually do all of this? Well, let's illustrate that with an mmap example!

#### An example: Mmap()

We have a dedicated blog post to mmap and page fault handling, so I recommend you check that out as a next step. But, to illustrate why this VMA design may be helpful - let's say you memory map a file to virtual memory (using the mmap() system call) of size 1  GiB. Reading it traditionally would mean that you would need to just stream the entire thing in, which is obviously inefficient.

So, we have our VMAs (and demand paging) which allow us to achieve this exactly. With our VMA system, we can:
1. Allocate one single VMA, from 0 to 1GiB
2. Not allocate any actual frames up front, meaning we do not take up any of physical memory yet
3. When accessed, we page fault - but it's okay! We can simply look at our VMA and, by our offset into the VMA, figure out what part of the file we should bring in!

Now, we have taken a 1 GiB file and made it so that we only actually take up 1 GiB of physical memory if we really need to!

For a more thorough breakdown of all of this, please take a look at the dedicated blog, and, if you're interested, read the source code (including about 1300 lines of testing code) which can be found [here](https://github.com/TAOS-Labs/TAOS/blob/main/kernel/src/memory/mm.rs)

## Conclusion

I hope that this has been an entertaining and educational foray into virtual memory. VM is my personal favorite part of systems, and I hope that I was able to at least somewhat convince you that it is interesting. 

There are many things that were not mentioned or elaborated on for brevity's sake, including but not limited to: fork(), loading a process, page permissions, COW pages, and page fault handling.

If you have any specific questions about my kernel, please do not hesitate to reach out to me at kshitij@utexas.edu

Thank you for reading!
