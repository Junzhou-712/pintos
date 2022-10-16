<div align="center">U<small>SER PROGRAM</small></div>
<div align="center">---- Background ----</div>

Before we begin with Project 2, we need remove the function of priority donation.  
In Pintos, every user program is run by a process. In modern os, a single thread can run multiple processes; however in Pintos, every thread will only run one and only one process.
```c

/* userprog/process.h */
```