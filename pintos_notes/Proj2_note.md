<div align="center">U<small>SER PROGRAM</small></div>
<div align="center">---- Background ----</div>

Before we begin with Project 2, we need remove the function of priority donation.  
In Pintos, every user program is run by a process. In modern os, a single thread can run multiple processes; however in Pintos, every thread will only run one and only one process.
This filename is "raw" in the sense that it contains both the executable name and the arguments, for example:
user_program arg1 arg2 arg3 arg4
Which means when a thread is created with thread_create in this function to run the user program, you will notice that the thread is named the raw filename:
tid = thread_create(file_name, PRI_DEFAULT, start_process, fn_copy);
> Write each argument (including the executable name) in reverse order, as well as in reverse for each string, to the stack. Remember to write a \0 for each argument. memcpy will come in handy here.
Write the necessary number of 0s to word-align to 4 bytes. A little modulus math and memset will get the job done.
Write the last argument, consisting of four bytes of 0’s.
Write the addresses pointing to each of the arguments. You’ll need to figure out how to reference the addresses after writing all the arguments. These are char* s.
Write the address of argv[0]. This will be a char**.
Write the number of arguments (argc). Make sure that this spans over 4 bytes.
Write a NULL pointer as the return address. This will be a void*.
```c
#ifndef USERPROG_PROCESS_H
#define USERPROG_PROCESS_H

#include "threads/thread.h"

/* utilize strtok_r() (lib/string.h) to implement the parameters passing instead of raw filename */
tid_t process_execute (const char *file_name);
int process_wait (tid_t);
void process_exit (void);
void process_activate (void);

#endif /* userprog/process.h */
```
In process.c, we can find start_process function call load() for the file, load attempt to execute the file and set up the stack.
Except that, there're also some intresting function we need to analyze.