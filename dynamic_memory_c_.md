# 🧠 Dynamic Memory in C: Understanding `malloc()`, `calloc()`, and `free()`

The importance of good memory management cannot be overstated when creating dependable and expandable programs. The C programming language has two methods for managing memory: static and dynamic allocation of memory. Static memory allocation occurs at the time the executable file is created, while dynamic memory allocation enables a program to request memory while it is executing.

This flexibility allows developers to manage their programs in an optimal way and increases the program's ability to adapt and perform effectively since developers may not know at development time the exact amount of memory required by their program.

What does it mean?

Dynamic Memory Allocation is when memory is allocated while a program is being run. Unlike static memory allocation, which is done at compile-time, dynamic memory allocation can be adjusted as needed, so that the value of memory can change depending on user input or conditions during runtime.

Example:

int x = 5; // Allocated at compile-time

Dynamic Memory Allocation Functions

There are three main functions provided by the C programming language for performing Dynamic Memory Management:

1) malloc() - Allocates memory
2) calloc() - Allocates and initializes memory
3) free() - Free memory allocated using malloc()/calloc().

The functions shown in the Library, are as follows: 

Using malloc().

The malloc() function is used for allocating memory. We specify how much memory we want to allocate by specifying how many bytes we would like to get allocated.

For example, we want to create an array of 5 integers:
int arr = (int) malloc(5 * sizeof(int));

Key Points: 
* Allocates memory for 5 integers.
* Returns a pointer to the allocated memory.
* Does not initialize memory (there are no 0s within the array until it is initialized). 
* Returns NULL if there are no memory left to allocate.

In general, the malloc() function is most useful when speed of memory allocation is important and there is no need to initialize memory values.

Usage of the calloc() function.

The calloc() function is similar to malloc(); however, it not only allocates memory but also initializes each value to zero.

For example: We want to create an array of 5 integers:
int arr = (int) calloc(5, sizeof(int));

Key Points:
* Requires a number of elements and the sizeof each of the element.
* Initializes memory to zero.
* Useful when values need to be initialized to default values.

Unlike the malloc() function, calloc() provides for use of predictable and cleanly used memory.

Usage of the free() function.

The free() function is used to de-allocate, or "release", dynamically allocated memory back to the system.

Generally just use the free() function to return the dynamically created block of data back to the system.

free(arr);
arr = NULL;

Here are a few key points to remember:
- Avoid memory leaks.
- Always make sure to free any dynamically created block when it is no longer needed.
- Setting the pointer to NULL after freeing it will prevent any program from mistakenly using or referencing that block of memory.

Failure to free blocks of dynamically created memory can have a negative effect on performance and cause the program to crash.

Memory Functions Comparison

Type        Instantiated        No. of parameters        Purpose
malloc        NO                       1                              to allocate quickly
calloc        YES (to Zero)      2                              to allocate cleared memory
free          N/A                   1                              to free from system memory

Consequences of Improper Use of Dynamic Memory

Developers must pay very close attention to their use of dynamic memory because the consequences can be serious.

- Using uninitialized memory results in unpredictable behaviour.
- Memory leaks occur when unused memory has not been released.
- Double Free Error occurs when trying to release the same block twice.
- Not checking for NULL pointer issues occurrence can happen if you've not checked the results from allocation.

It is extremely important for any developer to be very careful with their use of dynamic memory.

Performing memory management correctly is critical to the performance of any programming language. Here's a list of best practice guidelines for managing memory:

- After allocating memory, always verify that your pointer is not NULL.
- Free each piece of memory you use when you are done.
- Do not access memory that has already been freed.
- Use calloc() when you need to initialize the memory.
- Track all allocated memory blocks.

Dynamic memory allocation in C is one of the most powerful capabilities of the language. Dynamic memory allows you to have maximum flexibility and unrestricted usage of memory at run time.

While it allows programmers to maximize their program's performance, dynamic memory requires careful programming. Improperly using malloc(), calloc() and free() can lead to untraceable bugs in your programs. Understanding how to use these functions correctly can help you produce efficient and safe programs.