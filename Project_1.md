# Project 1: Linux System Call Implementation & Kernel Stack

## 1. Executive Summary
* **Environment**: WSL2 (Ubuntu 22.04 LTS), Custom Linux Kernel `6.6.36`
* **Objective**: Understand the end-to-end flow of Linux system calls and implement custom system calls to manage a thread-safe stack data structure in kernel memory space.

---

## 2. Linux System Call Flow & Control Routine

When a user-space application requests a service requiring kernel privileges (e.g., accessing hardware or privileged kernel structures), execution switches from user mode to kernel mode via system calls.

```
[ User Space ]
  call_my_stack (User App) 
     └── syscall(335, val)
           │
           ▼ (Trigger Software Interrupt / Trap: User -> Kernel Mode)
[ Kernel Space ]
  entry_SYSCALL_64
     └── System Call Table Lookup (syscall_64.tbl)
           │
           ▼ (Invoke mapped function via Index 335)
  sys_os2026_push()
     ├── Allocate kernel memory (kmalloc)
     ├── Acquire Mutex Lock (os2026_stack_lock)
     ├── Modify kernel dynamic stack
     └── Release Lock & Return status code to User Space
```

1. **User Mode Execution**: The user application calls `syscall()` provided by `glibc`. Arguments and the system call index number are loaded into registers (`rax`, `rdi`, etc.).
2. **Mode Switch**: Executing the `syscall` instruction causes a CPU trap, switching execution from Ring 3 (User Mode) to Ring 0 (Kernel Mode) and transitioning to the kernel stack.
3. **Table Lookup**: The kernel checks the `syscall_64.tbl` using the index number to resolve the entry function address (e.g., `sys_os2026_push`).
4. **Execution & Return**: The kernel executes the mapped C function with privileged instructions. Upon completion, the result/error code is stored in the return register (`rax`), restoring registers and handing control back to user space.

---

## 3. Architecture & Key Implementation Mechanics

*Note: Source code has been omitted to respect copyright guidelines.*

### 3.1 Kernel Stack Data Structure
The kernel-managed stack is implemented as a **singly linked list**:
* **Node Structure**: Contains an integer payload (`number`) and a pointer (`next`) to the subsequent node.
* **Stack Head**: A global static pointer `top` represents the top of the stack, initialized to `NULL`.

### 3.2 Synchronization & Thread Safety
* Since system calls can be invoked concurrently across multiple processes or CPU cores, modifying global shared memory (`top`) without protection introduces critical race conditions.
* **Primitive Used**: A static kernel mutex (`DEFINE_MUTEX(os2026_stack_lock)`) enforces mutual exclusion during data structure modifications and stack state traversals.

### 3.3 Implemented System Calls
1. **`sys_os2026_push(int val)` (Syscall #335)**
   * **Memory Management**: Allocates memory in kernel space using `kmalloc(sizeof(...), GFP_KERNEL)`.
   * **Logic**: Locks mutex, connects new node as the new `top`, logs updated stack state via `printk()`, and unlocks.
2. **`sys_os2026_pop()` (Syscall #336)**
   * **Logic**: Locks mutex, validates empty stack state (returns negative error on empty), retrieves node payload, updates `top`, frees previous node memory via `kfree()`, and returns popped value.
3. **`sys_os2026_clear()` (Syscall #337)**
   * **Logic**: Iterates through the linked list, deallocates all nodes via `kfree()`, counts removed elements, and resets `top` to `NULL`.

---

## 4. Key Takeaways & Lessons Learned

1. **Kernel Space vs. User Space Restrictions**:
   * User-level C libraries (`stdlib.h`, `stdio.h`, `malloc/free`) are unavailable inside kernel space. Kernel-specific APIs (`<linux/slab.h>`, `kmalloc`, `kfree`, `printk`) must be utilized.
2. **Concurrency Control in Kernel Development**:
   * Shared kernel dynamic structures require explicit locking primitives to prevent race conditions during concurrent syscall invocations.