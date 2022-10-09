# Project 1
- Task 1: Recoginize functions structure  
the first chanllenging function that I confronted is switch_threads in switch.S. As for this function, I am gonna leave the snippet and my own comments for reference. 
```x86asm
switch_threads:
	# Save caller's register state.
	# According to struct switch_threads_frame defined in switch.h. Registers' stautus pushed in advance.
	# in size.
	pushl %ebx
	pushl %ebp
	pushl %esi
	pushl %edi

	# Get offsetof (struct thread, stack).
    # thread_stack_ofs is a global variable and we mov it to register
.globl thread_stack_ofs
	mov thread_stack_ofs, %edx

    # Saving the context in %eax and %ecx
	# Save current stack pointer to old thread's stack, if any.
	movl SWITCH_CUR(%esp), %eax
	movl %esp, (%eax,%edx,1)

	# Restore stack pointer from new thread's stack.
	movl SWITCH_NEXT(%esp), %ecx
	movl (%ecx,%edx,1), %esp


	# Restore caller's register state.
	popl %edi
	popl %esi
	popl %ebp
	popl %ebx
    # ret is %eax, which stored the cur stack pointer (that's why we assign prev = switch_threads)
        ret
.endfunc
```
<div align="center">A<small>LARM CLOCK</small></div>
<div align="center">---- DATA STRUCTURES ----</div>

```c
/* Block ticks (control thread_sleep). */
unsigned int block_ticks >> struct thread_t
/* Check thread's status. If thread is blocked and its block ticks is greater than 0, minus 1 and check whether unblock it*/
void 
thread_check_blocked(struct thread *t, void *aux UNUSED) >> thread.h 
```
<div align="center">---- ALGORITHMS ----</div>

```c
/* Sleeps for approximately TICKS timer ticks.  Interrupts must be turned on. */
void
timer_sleep (int64_t ticks) 
{
  if (ticks <= 0) return; //edge case
  ASSERT (!intr_context());
  ASSERT (intr_get_level () == INTR_ON);
  enum intr_level old_level = intr_disable(); // Prevete race conditions like multiple threads call or timer interrupt happens during timer_sleep()
  thread_current()->block_ticks = ticks; // Assign thread block_ticks to ticks
  thread_block();
  intr_set_level(old_level);
}
```
<div align="center">P<small>RIORITY SCHEDULING</small></div>
<div align="center">---- DATA STRUCTURES ----</div>

```c
/* a compare function return true if a->priority > b->priority */  
bool
thread_more_priority >> list.h
```
However, the function above only solved the situation that scheduler insert a new thread to the list. If a thread call a function thread_set_priority to change its own priority we cannot handle this situation to release the resource and block the thread (if its priority is less than anyone in the ready list). So we add a function **thread_yiled()** following the thread priority changed.  
[Priority Inversion](https://en.wikipedia.org/wiki/Priority_inversion) happened if we do not handle the shared resource is released properly when yielding. Thus, there are the snippets of priority inversion solution.  



<div align="center">---- ALGORITHMS ----</div>