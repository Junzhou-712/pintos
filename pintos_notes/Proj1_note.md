# Project 1
- Task 1: Recoginize functions structure  
the first chanllenging function that I confronted is switch_threads in switch.S. As for this function, I am gonna leave the snippet and my own comments for reference. 
```
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