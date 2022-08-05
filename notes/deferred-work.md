# Deferred Work

Reference: https://linux-kernel-labs.github.io/refs/heads/master/labs/deferred_work.html

- Deferred work is a class of kernel facilities that allow to schedule code to be executed at a later timer
- Summarizing we have two categories:
  - Running in interrupt context, also known as **bottom-half** (BH): Softirqs, Tasklets, Timers
  - Running in process context: Workqueues, Kernel Threads. Kernel Threads are not strictly speaking deferred work in themselves, but are used to complement deferred work mechanisms
- When running in interrupt context we have the following restrictions:
  - Can't transfer data from or to user space (they don't execute in the context of a process)
  - Can't sleep (e.g., call `wait_event`, allocate memory without `GFP_ATOMIC`, acquire semaphores or mutexes, etc.)
  - Can't call `schedule`

## Types of deferred work

### Softirqs

- Can't be used by device drivers, reserved for various kernel subsystems
- They run in **interrupt context**, so they can't call blocking functions

### Tasklets

- They run in **interrupt context**, so they can't call blocking functions
- Are used to schedule some action to be executed later, but can't be used to schedule at a *specific* time

### Timers

- They run in **interrupt context**, so they can't call blocking functions
- Used to schedule a function to execute a specific time
- Unit of time is jiffies

### Workqueues

- Used to schedule actions to run in **process context**
- Can be either scheduled at a specific time, or not (`work_struct` vs `delayed_work`)

### Kernel Threads

- Basically a thread that runs only in kernel mode, and has no user address space or other user attributes
- Used to run kernel code in **process context**
- Used with work queues
- They are not technically deferred work

## Synchronization

- When synchronizing code running in process context and interrupt context, we need to use special locking primitives:
  - In interrupt context, we can use the usual spinlock operations (`spin_lock`, `spin_unlock`)
  - In process context, we have to disable bottom-half handlers in the current processor besides acquiring a lock. This can be done with `spin_lock_bh` and `spin_unlock_bh`
- This is necessary to avoid creating a deadlock if we acquire a lock in process context, but then a BH handler that also has to acquire the lock is scheduled in the same CPU (for example, because a timer fires, etc.)

