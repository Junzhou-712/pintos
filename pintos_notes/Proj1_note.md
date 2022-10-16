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
bool
lock_more_priority >> list.h
bool
cond_sema_more_priority >> list.h
struct
semaphore_elem >> synch.h
```
However, the function above only solved the situation that scheduler insert a new thread to the list. If a thread call a function thread_set_priority to change its own priority we cannot handle this situation to release the resource and block the thread (if its priority is less than anyone in the ready list). So we add a function **thread_yiled()** following the thread priority changed.  
[Priority Inversion](https://en.wikipedia.org/wiki/Priority_inversion) happened if we do not handle the shared resource is released properly when yielding. Thus, there needs to have a design to solve priority inversion.  
PintOS suggests priority donation to solve the share resource contension conflict, which basically improve the priority of the thread holds the lock (kinda like donation from the waiting list), when this thread's priority is less than the waiting one. When the thread releases the lock, it will convert its priority to original value.  
To implement the functionality of priority donation, I added several functions for PintOS.
```c
/* Donate max_priority to the resource held the lock/sema */
void
thread_donate_priority(struct thread *donated)
{
  enum intr_level old_level = intr_disable();
   thread_update_priority (donated);
   if(donated->status == THREAD_READY) {
    list_remove(&donated->elem);
    list_insert_ordered (&ready_list, &donated->elem, thread_more_priority, NULL);
   }
  intr_set_level(old_level);
}

/* Update thread priority recursively */
void
thread_update_priority(struct thread *t)
{
  enum intr_level old_level = intr_disable();
  int max_priority = t->origin_priority;
  int lock_priority;
  if(!list_empty(&t->locks)) {
    list_sort(&t->locks, lock_more_priority, NULL);
    lock_priority = list_entry(list_front(&t->locks), struct lock, elem)->max_priority;
    if(lock_priority > max_priority) {
      max_priority = lock_priority;
    }
  }
  t->priority = max_priority;
  intr_set_level(old_level);
}
```

<div align="center">---- ALGORITHMS ----</div>
Priority donation's logic could be revealed from the several test cases. the general rules are:  

  - Several Priority Queues
    - semaphore->waiter
    - struct condition
  - Combine thread_yield() and thread_update_priority() (implemented above) to donate the priority to the thread using the shared resource.
  - Nested donation: a thread's donation must be donated to not only its recipient, but also the thread its recipient is waiting on, which means that it is a recursive donation.
  - Donation chain: the donation must propagate through an arbitrary number of nested donations.  
When I tried to figure out these test cases, the test case "test_priority_donate_sema" stuck my progress. If we gonna solve this test case, the first thing is analyze the test source code.
```c
void
test_priority_donate_sema (void)
{
  struct lock_and_sema ls;

  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);

  /* Make sure our priority is the default. */
  ASSERT (thread_get_priority () == PRI_DEFAULT);

  lock_init (&ls.lock);
  sema_init (&ls.sema, 0);
  // sema assign to 0 initially
  thread_create ("low", PRI_DEFAULT + 1, l_thread_func, &ls);
  thread_create ("med", PRI_DEFAULT + 3, m_thread_func, &ls);
  thread_create ("high", PRI_DEFAULT + 5, h_thread_func, &ls);
  // sema_up V operation release resource awake low thread <- 4
  sema_up (&ls.sema);
  msg ("Main thread finished.");
}

static void
l_thread_func (void *ls_)
{
  // lock_and_sema is a combination of lock and semaphore
  struct lock_and_sema *ls = ls_;

  lock_acquire (&ls->lock);
  msg ("Thread L acquired lock.");
  // low thread tries to acquire the lock and P operation blocks low thread <- 1 
  // To satifiy this task, low thread was donated by high thread <- 4
  sema_down (&ls->sema);
  msg ("Thread L downed semaphore.");
  // low thread released the lock awake high thread <- 5
  lock_release (&ls->lock);
  // low thread output the msg <- 8
  msg ("Thread L finished.");

}

static void
m_thread_func (void *ls_)
{
  struct lock_and_sema *ls = ls_;
  // P operation blocks Med thread <- 2
  sema_down (&ls->sema);
  // med thread is awoken by 6 and output the msg <- 7
  msg ("Thread M finished.");
}

static void
h_thread_func (void *ls_)
{
  struct lock_and_sema *ls = ls_;
  // High thread tries to acquire the lock and P operation blocks high thread <- 3
  lock_acquire (&ls->lock);
  msg ("Thread H acquired lock.");

  sema_up (&ls->sema);
  lock_release (&ls->lock);
  // high thread V operation release the lock <- 6
  msg ("Thread H finished.");
}
```
The sequence of a series of events is:  
1. low thread tries to acquire the lock and P operation blocks low thread <- 1 
2. P operation blocks Med thread <- 2
3. High thread tries to acquire the lock and P operation blocks high thread <- 3
4. sema_up V operation release resource awake low thread <- 4
5. low thread released the lock awake high thread <- 5
6. high thread V operation release the lock <- 6
7. med thread is awoken by 6 and output the msg <- 7
8. low thread output the msg <- 8  
   According to the sequence I designed the relevant functions including lock_release(), lock_acquire(), sema_up(), sema()_down().
```c
void
sema_up (struct semaphore *sema) 
{
  enum intr_level old_level;

  ASSERT (sema != NULL);

  old_level = intr_disable ();

  if (!list_empty (&sema->waiters))  {
    list_sort(&sema->waiters, thread_more_priority, NULL);
    thread_unblock (list_entry (list_pop_front (&sema->waiters),struct thread, elem));
  }
  sema->value++;
  thread_yield();
  intr_set_level (old_level);
}

void
sema_down (struct semaphore *sema)
{
  enum intr_level old_level;

  ASSERT (sema != NULL);
  ASSERT (!intr_context ());

  old_level = intr_disable ();
  while (sema->value == 0)
    {
      list_insert_ordered (&sema->waiters, &thread_current ()->elem, thread_more_priority, NULL);
      thread_block ();
    }
  sema->value--;
  intr_set_level (old_level);
}

void
lock_release (struct lock *lock) 
{
  ASSERT (lock != NULL);
  ASSERT (lock_held_by_current_thread (lock));
  if(!thread_mlfqs) {
    thread_remove_lock(lock);
  }
  lock->holder = NULL;
  sema_up (&lock->semaphore);
}

void
lock_acquire (struct lock *lock)
{
  struct thread *cur_thread = thread_current ();
  struct lock *l;
  enum intr_level old_level;

  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (!lock_held_by_current_thread (lock));

  if (lock->holder != NULL && !thread_mlfqs)
  {
    cur_thread->waiting_locks = lock;
    l = lock;
    while (l && cur_thread->priority > l->max_priority)
    {
      l->max_priority = cur_thread->priority;
      thread_donate_priority (l->holder);
      l = l->holder->waiting_locks;
    }
  }

  sema_down (&lock->semaphore);

  old_level = intr_disable ();

  cur_thread = thread_current ();
  if (!thread_mlfqs)
  {
    cur_thread->waiting_locks = NULL;
    lock->max_priority = cur_thread->priority;
    thread_hold_the_lock (lock);
  }
  lock->holder = current_thread;

  intr_set_level (old_level);
}
```
<div align="center">A<small>DVANCED SCHEDULER</small></div>
<div align="center">---- DATA STRUCTURES ----</div>

- Nice value 
  Function: int thread_get_nice (void)
    Returns the current thread's nice value.
  Function: void thread_set_nice (int new_nice)
    Sets the current thread's nice value to new_nice and recalculates the thread's priority based on the new value.If the running thread no longer has the highest priority, yields.  
-  There's a formula: 
  $$recent\_cpu = (2*load\_avg)/(2*load\_avg + 1) * recent\_cpu + nice$$
where load_avg is a moving average of the number of threads ready to run (see below). If load_avg is 1, indicating that a single thread, on average, is competing for the CPU, then the current value of recent_cpu decays to a weight of .1 in ln(.1)/ln(2/3) = approx. 6 seconds; if load_avg is 2, then decay to a weight of .1 takes ln(.1)/ln(3/4) = approx. 8 seconds. The effect is that recent_cpu estimates the amount of CPU time the thread has received "recently," with the rate of decay inversely proportional to the number of threads competing for the CPU.  
Before implementing the mlfqs, we should figure out PintOS does not support float number computing. The table below summarizes how fixed-point arithmetic operations can be implemented in C. In the table, x and y are fixed-point numbers, n is an integer, fixed-point numbers are in signed p.q format where p+q = 31, and f is 1<<q:

|||
|-|-|
|Convert n to fixed point:|	n * f|
|Convert x to integer (rounding toward zero):|	x / f|
|Convert x to integer (rounding to nearest):	|(x + f / 2) / f if x >= 0,(x - f / 2) / f if x <= 0.|
|Add x and y:|	x + y|
|Subtract y from x:	|x - y|
|Add x and n:|	x + n * f|
|Subtract n from x:|	x - n * f|
|Multiply x by y:| ((int64_t) x) * y / f|
|Multiply x by n:|	x * n|
|Divide x by y:|	((int64_t) x) * f / y|
|Divide x by n:|	x / n|

"thread/fixedpoint.h" will implement these arithmetics. I define FP_SHIFT_AMOUNT as the decimal part.

<div align="center">---- ALGORITHMS ----</div>
mlfqs is the shorts for "multilevel feedback queue scheduling"