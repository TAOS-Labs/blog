+++
title = 'Events and Scheduling'
date = 2025-05-02T20:56:24-05:00 
draft = false
author = 'Kiran Chandrasekhar' 
+++

# A non-preemptible kernel

Preemption; that is, the forceful stopping of a unit of code to allow another to be scheduled, is a critical component of any useful Operating System. Without it, arbitrary user processes could lock down a CPU core for as long as they like; moreover, even nice processes would have to be constantly aware of how long they are running and yield control when needed.

However, in the kernel, we can possibly get away with not preempting. So long as we carefully write code to frequently block or yield, then our untrusted code can run freely. How can this be implemented, what are the benefits, and what exactly does this entail?

### Events

We define an "event" in the kernel as a unit of code corresponding to some task. For example, a request to setup a DMA buffer may be categorized as a kernel event. Each event is largely self-contained, though perfectly capable of communication with any other through the singular kernel heap or global memory. They can voluntarily yield or perform blocking operations, thus ensuring that each event knows exactly when and where it may lose control of the CPU. In these spots the event can chose to save only the important components of its state, thus intelligently reducing the cost of a task switch.

In our kernel, these events are implemented using Rust's async/await feature. This feature allows functions to be marked as "async", which causes the compiler to transform them into functions that return a Future. These futures may then be awaited upon, where the code inside is run and the result is evaluated; in the event that the future must block for whatever reason, the calling function is notified. This creates a hierarchy of futures, with the topmost future resolving the event itself. This topmost future may then be polled by the event scheduler, returning either Poll::Ready if it is complete or Poll::Pending if it must be rescheduled anew. In the event that it must be rescheduled, only the necessary state for progress is saved, stashed cleanly and robustly onto the kernel heap for access later, creating a state machine of sorts.

```rust
/// Describes a future and its scheduling context
pub struct Event {
    eid: EventId,
    pid: u32,
    future: SendFuture,
    rewake_queue: Arc<EventQueue>,
    blocked_events: Arc<RwLock<BTreeSet<u64>>>,
    priority: AtomicUsize,
    scheduled_timestamp: AtomicU64,
    completed: AtomicBool,
}
```

In this context, each event consists of a unique event id, a pid (0 if it is a kernel event), the topmost future to operate on, and then some references to scheduler data structures and metadata. The scheduler provides a two ways to create events: "schedule_kernel" to notify the scheduler of a new kernel event and "schedule_process" to notify the scheduler of a new user process. The former takes in a future, which it then wraps into an Event. 

This makes round-robin scheduling easy: when an event returns Poll::Ready, it may simply be removed from the event queue(s). If it returns Poll::Pending, it may simply be pushed onto said queue(s).

### Rust async/await

In userland Rust, async/await functions are supported via a runtime which schedules and executes the futures; e.g. Tokio. Of course, we have no such luxury in our own custom kernels. Thus, we will need to implement the necessary infrastructure for async/await. 

1. First and foremost comes a heap allocator. Without this kernel heap setup, futures will be unable to save their necessary state.
2. Next comes the ```Executor```. This functions as the Event Runner, polling events until they are ready. It will need to hold the pending events in some data structure, such as a simple queue. Further expansion is highly encouraged; we implement separate queues for blocking and pending events to avoid wasting cycles polling.
3. After this is a ```Waker```. Futures can and will notify their executor of a change in state via a registered waker. For example, a sleeping event can be awoken once the time has passed. In this situation, a waker should implement the wake_by_ref function to move an event from a blocked queue to a pending queue, ready to be rescheduled at a moment's notice.
4. Optionally, one may also implement a ```Spawner```. A spawner's job is just to schedule events to be run in the background and only awaited upon later.


### Benefits of no preemption

When preempting code, the state can be unbounded and completely arbitrary. As such, it is necessary to carefully save every necessary component of state, a rather costly endeavor. Furthermore this requires careful synchronization as code may be stopped at any point, not to mention the use of multiple kernel stacks as one switches between units of code in the kernel.

Without preemption, as mentioned above, we can use async/await language features to implement events. As these will always run until the next .await call, the possible state that needs to be saved is limited. Synchronization can also be somewhat relaxed; though parallel accesses still exist across cores, two concurrent events on the same core can avoid interleaved critical sections so long as there is no .await call in said critical sections. Furthermore, the kernel stack may then be limited to one per core, as events can simply build off of the existing stack (as their data is stored on the heap).

### User processes

Of course, one cannot skip out of implementing preemption, unless the kernel is only every executing a small subset of known, trusted processes. User processes may be considered events themselves, but with special markers marking them as non-preemptible (such as a distinct pid from the default for kernel events). Thus, each user process becomes an event which sets up and executes a process, like so:

```rust
pub async unsafe fn run_process_ring3(pid: u32) {
    // Sets up user stack for iretq and restores state from PCB
    resume_process_ring3(pid);

    loop {
        let process = {
            let process_table = PROCESS_TABLE.read();
            let Some(process) = process_table.get(&pid) else {
                return;
            };
            process.clone()
        };

        // Do not lock lowest common denominator
        let process = process.pcb.get();

        // Read PCB information to determine current process state + handling
        let arg1 = (*process).reentry_arg1;
        let reentry_rip = (*process).reentry_rip;
        let kernel_rsp = &mut (*process).kernel_rsp as *mut u64 as u64;
        let in_kernel = (*process).in_kernel;

        if (*process).state == ProcessState::Blocked || (*process).state == ProcessState::Ready {
            interrupts::disable();

            yield_now().await;
            if in_kernel {
                // Switch back to syscall stack
                unsafe {
                    asm!(
                        "push rax",
                        "push rcx",
                        "push rdx",
                        "call resume_syscall",
                        "pop rdx",
                        "pop rcx",
                        "pop rax",
                        in("rdi") arg1,
                        in("rsi") reentry_rip,
                        in("rdx") kernel_rsp as *mut u64,
                    );
                }
            } else {
                // Came from process (likely timer interrupt preemption)
                // No need to check any futures, can simply resume the process
                resume_process_ring3(arg1 as u32);
            }
        }
    }
}
```

Running the process itself requires much unsafe assembly to setup the iretq stack and populate registers from the PCB without clobbering. This logic is handled in the resume_process_ring3 function. After this, the current core context switches back to the user process. This is where things get tricky with async/await...

Normally, with kernel events directly invoking other kernel function calls, the compiler may statically know how to set up the futures state machine. However, here, where arbitrary user processes are running completely unaware of any sort of Rust runtime context, let alone the usage of async/await in the kernel, we cannot rely heavily on this. And yet, processes must yield Poll::Pending many times; e.g. on preemption or on invocation of a blocking syscall. This necessitates the use of additional PCB data; e.g. a reentry pointer which notifies the preemption/blocking where to return to in the broader run_process_ring3 event and one which notifies the runner event on where to return in the syscall handler or user process. These instruction pointers in Rust are best obtained as function pointers, as rustc will then handle most of the grunt work. This enables control flow magic to bypass the arbitrary series of function calls in between.

Of course, such control flow magic is highly volatile and bug-prone. Assuming a working implementation, one must also be wary of memory leaks. Rust will struggle to maintain understanding of when to free a Box'd pointer if the control flow is messed with in raw assembly.

To see examples of this, checkout the [process code](https://github.com/TAOS-Labs/TAOS/blob/main/kernel/src/processes/process.rs) and [blocking tools](https://github.com/TAOS-Labs/TAOS/blob/main/kernel/src/syscalls/block.rs). The latter enable the continued use of async/await in kernel code even when invoked from a process (such as a syscall). With the control flow magic, the process itself is elided in order to return a future directly to the runner event.

