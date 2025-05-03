# Handling Page Faults with MMAP


The page fault handler is one of the more complex parts of implementing a virtual memory system with mmap. However, there's surprisingly little information online about how to adequately handle all the different types of page faults you can get with mmap(). There are several different cases to handle depending on the type of mapping being used. Let's first review the different types of mappings that you must support when implementing mmap().


### Types of Mappings


- Anonymous Mappings
Anonymous mappings are not backed by any file. They simply represent a portion of memory that is zero-initialized. These regions of memory can be used for memory allocations by functions such as malloc that generally use mmap for larger allocations rather than using sbrk. This is done to help reduce the negative effects of memory fragmentation when large blocks of memory are freed but fragmented due to smaller allocated blocks lying in between them [[1]](#1).


- File-backed Mappings
File-backed mappings represent a portion of a file that is loaded into memory. Depending on the type of the mapping, changes made to this memory region may or may not be reflected back onto the file itself on disk. File-backed mappings are used to allow for more efficient access to file data since the files are loaded into memory. Furthermore, files are loaded on demand, meaning only the portion of the file that is trying to be accessed is actually loaded into memory. File-backed mappings are commonly used for loading shared object files for dynamic linking, loading executable files, and whenever a file needs to be read from memory rather than from disk.


There are also two different relevant sharing modes that can change the behavior of an anonymous or file-backed mapping.


- Private mappings
When a mapping is private it means that a process has its own isolated view of that mapped region of memory. When the region of memory is initially created, physical frames of memory are not immediately duplicated. Instead, they are loaded on-demand. Furthermore, when the kernel creates page table entries for the mapped pages of memory, they are always initially set to read only. When a process tries to write to a mapped page, it will then trigger a page fault due to a permissions violation, which is handled through Copy-on-Write (COW) semantics which will be discussed further in the page fault section.


The implementation differs between anonymous and file-backed mappings. For an anonymous mapping, COW comes into play when the process forks. The child and parent process initially share the anonymous pages, but if either process writes to a page, a private copy is created for the writer. For a file-backed mapping, since the process has its own isolated view of the mapped memory region, all modifications are unique to that process and are never propagated back.


- Shared mappings
When a mapping is shared it means that any change to that mapped memory region is visible to all processes that map the same file/physical memory. Since the mappings are shared, there are now COW semantics - when a modification is made, the modifications directly affect all shared pages. These changes are also immediately visible to all processes that map the same region.


For file-backed shared mappings, this requires a page cache integration, where modified pages are tracked so that the kernel knows which pages need to be written back. For anonymous mappings, sharing mostly happens through the fork system call, via inheritance of the parent processes' pages. Mapped memory persists while at leats one process maintains the mapping, requiring a reference count of mapped pages of memory across multiple processes.


### The Page Cache

The page cache is a vital part of managing file-backed mappings with mmap. In our kernel, the page cache is defined as follows:


```rust
type PageCache = Mutex<BTreeMap<u32, Arc<Mutex<BTreeMap<usize, Page<Size4KiB>>>>>>;
```


The outer BTreeMap maps inode numbers to inner BTreeMaps, creating a unique mapping between files and flie mappings. The inner BTreeMap maps file offsets to pages of memory. This way, we have a per-file data structure that can lazy-load pages of memory that map parts of a file. The offset dictates which portion of the file is mapped by the relevant page. When a process tries to access a file-backed mapping for the first time, the kernel will index into the page cache, first by the inode number of the relevant file, and then by the position within the file that is being accessed (this is stored as metadata in a File struct that represents a file). If the kernel is able to find the entry, then the operating system can simply map the already existing page in this process' address space. If not, a new entry can be created after reading the relevant page of data for the file from disk.
Lazy Loading and Page Faults


The key to an implementation of mmap() is lazy loading. When a process calls mmap(), no memory is immediately allocated for anonymous mappings or read from disk for file-backed mappings. The mapping is simply recorded as a virtual memory area, which contains the start and end of the memory region, the backing (see the post exploring the data structures used in our mmap() implementation), and other metadata related to permissions. When a process tries to access data, it will immediately page fault since no memory is actually allocated. This way, only when memory is actually needed will it ever be allocated or read from disk.


The Page fault handler handles six different types of page faults that can occur.


```rust
pub enum FaultOutcome {
    /// MAP_ANONYMOUS | MAP_SHARED
    SharedAnonMapping {
        /// page containing faulting address
        page: Page<Size4KiB>,
        /// this process' page table mapper
        mapper: OffsetPageTable<'static>,
        /// The backing of frames for the mmap'ed area that contains the faulting address
        chain: Arc<VmaChain>,
        /// the page table flags for the page containing the faulting address
        pt_flags: PageTableFlags,
    },
    /// MAP ANONYMOUS | (MAP_SHARED or MAP_PRIVATE)
    NewAnonMapping {
        /// page containing faulting address
        page: Page<Size4KiB>,
        /// this process' page table mapper
        mapper: OffsetPageTable<'static>,
        /// backings to use
        backing: Arc<VmAreaBackings>,
        /// the page table flags for the page containing the faulting address
        pt_flags: PageTableFlags,
    },
    /// MAP_FILE | MAP_SHARED
    SharedFileMapping {
        /// page containing faulting address
        page: Page<Size4KiB>,
        /// this process' page table mapper
        mapper: OffsetPageTable<'static>,
        /// the offset within the file that we had a page fault at
        offset: u64,
        /// the page table flags for the page containing the faulting address
        pt_flags: PageTableFlags,
        /// the file descriptor for the file that we had a page fault at
        fd: usize,
    },
    /// MAP_FILE | MAP_PRIVATE
    PrivateFileMapping {
        /// page containing faulting address
        page: Page<Size4KiB>,
        /// this process' page table mapper
        mapper: OffsetPageTable<'static>,
        /// the offset within the file that we had a page fault at
        offset: u64,
        /// the page table flags for the page containing the faulting address
        pt_flags: PageTableFlags,
        /// the file descriptor for the file that we had a page fault at
        fd: usize,
    },
    CopyOnWrite {
        /// page containing faulting address
        page: Page<Size4KiB>,
        /// this process' page table mapper
        mapper: OffsetPageTable<'static>,
        /// the page table flags for the page containing the faulting address
        pt_flags: PageTableFlags,
    },
    SharedPage {
        /// page containing faulting address
        page: Page<Size4KiB>,
        /// this process' page table mapper
        mapper: OffsetPageTable<'static>,
        /// the page table flags for the page containing the faulting address
        pt_flags: PageTableFlags,
    },
    // Address is already mapped - probably an error
    Mapped,
}
```


The first four only happen if the page containing the faulting address was never mapped for the current process. The latter two only happen if the page is actually mapped.


The first is when you have a shared anonymous mapping. In this case, you have multiple processes that share a region of anonymous memory. The region has already been allocated and mapped by at least one other process, but the current process has not yet added an entry for it, since everything is lazy loaded. The solution is to simply take the physical frame of memory associated with the shared page, and create another mapping to it for the current process. This case can only happen when you have an anonymous shared mapping.


The second is when you have a new anonymous mapping. At this point, since this is a completely new frame, it does not matter whether or not another process is going to share it or not. We page fault simply because there is no mapping for this memory address, and there was never one created. To fix this, we allocate and map a new frame and add it to the relevant backing based on the virtual memory area relevant to the faulting address. This case is only for anonymous mappings but can happen when you have either a private or shared mapping, since it just handles the initial frame mapping and allocation.


The third is when you have a file-backed mapping that is shared between multiple processes. The specific offset within the file has not been mapped to this current process. There are two possibilties for this type of page fault. The specific offset might be or might not be in the page cache, depending on whether a different process had already accessed this section of the file. We index into the page cache to find out whether the file offset has already been mapped. If it was not, we add a new entry, and once we are guaranteed that there is an entry in the page cache, we can take associated frame from the page cache and create a new mapping to it for the current process.


The fourth is when you have a private file-backed mapping. The implementation for this is similar to that of a shared file-backed mapping. The key difference is that since it is private instead of shared. we actually want it to be COW. That way whenever the process tries modifying the memory region, it is given its own copy to modify. Beyond that change, the same steps apply to handle this type of page fault as the shared file-backed mapping.


These next two fault outcomes happen for already mapped pages.


The fifth fault is when you have a COW fault. A COW fault occurs when the memory region is shared and has write permissions. Furthermore, the page fault happened because the process tried writing to a spot in memory. We simply take the data present in the old shared page, allocate and map a new one, and copy the data over.


The sixth and final fault cause happens when you do not have a COW fault but the page fault was caused by a write. We simply update the permissions for the page containing the faulting address to give the faulting process write permissions.


For a detailed look at the code required to implement the page fault handler, take a look at the source code for the TAOS kernel. The most relevant parts are the [page fault handler](https://github.com/TAOS-Labs/TAOS/blob/main/kernel/src/interrupts/idt.rs#L148) itself and the implementations of the fault cause determiner and the fixes for each possible page fault variation located [here](https://github.com/TAOS-Labs/TAOS/blob/main/kernel/src/memory/page_fault.rs).

# References

<a id="1">[1]</a>
https://www.linuxjournal.com/article/6390
