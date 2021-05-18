# The Stack and the Heap

In this tutorial, I'll guide you through different standard memory regions, and where different sorts of variables are stored.

To keep things simple, there are two types:

+ A signed integer: `int`
+ A pointer to a signed integer: `*int`

Also:

+ I'm assuming that the code has access to the entire memory space, and that there's no memory virtualisation or memory mapping.
+ I'm using a calling convention where function parameters and return values are places on the stack. In some calling conventions, low registers are used instead.
+ The compiled code is horribly unoptimised. In reality, the compiler will make intelligent use of memory locations to avoid unnessary movement of values.

## The Data Segment

The Data Segment stores global variables, and the allocation is chosen by the compiler, i.e. each differently named variable is given a different memory address.


    int a, b, c;

    void main() {
        a = 1;
        b = 2;
        c = a + b;
        printf("%d + %d = %d", a, b, c);
    }

The memory layout is thus as follows:

| Address | Variable |
| ------: | -------: |
|       2 |        c |
|       1 |        b |
|       0 |        a |

Note that:

+ Functions can only changed global variables; There's no concept of local variables.
+ There's no protection for a function changing a variable and thus overriding a wanted value.
+ Recursive functions are prohibited, as there's no way of recording the return address

This also means that recursive functions are impossible.

    int a, b, c;

    void set_a() {
        a = 1;
    }

    void set_b() {
        b = 2;
    }

    void corrupt_data() {
        a = 6;
        b = 8;
    }

    void main() {
        set_a();
        set_b();
        corrupt_data();
        printf("%d + %d = %d", a, b, c);
    }

As such, another memory region is declared: the stack

## The Stack

At the high level, the stack is a data structure where the last value to be placed on the stack is the first one two be removed.

A the low level, the stack consists of an allocated memory region, and two special registers: the stack register (SR) and the frame register (FR).

By most conventions, the values of these registers are initally set to the total memory size, i.e. the allocated memory region is at the end of the memory.

As both the stack register and frame register store addresses in memory, their values are ofter referred to as the stack pointer and frame pointer respectively.

In the following examples, I'll assume that the total memory size is 256 `int`s.

The stack stores function parameters, function return values, and local variables.

    int get_a() {
        return 2;
    }

    int get_b() {
        return 5;
    }

    int get_c() {
        return 8;
    }

    int add_two_numbers(int a, int b) {
        int c;
        c = a + b;
        return c;
    }

    void main() {
        int a, b, c, total;

        a = get_a();
        b = get_b();
        c = get_c();

        total = add_two_numbers(a, b);
        total = add_two_numbers(total, c);

        printf("%d + %d + %d = %d", a, b, c, total);
    }

At the start of `main`, the memory layout is as follows:

|       Address |          Variable |
| ------------: | ----------------: |
| SR = FR = 256 | \<end of memory\> |
|           255 |                 ? |
|           254 |                 ? |
|           ... |                 ? |

    int a, b, c, total;

Four variables are declared, so the stack pointer is decremented by 4:

|  Address |          Variable |
| -------: | ----------------: |
| FR = 256 | \<end of memory\> |
|      255 |         total = ? |
|      254 |             c = ? |
|      253 |             b = ? |
| SR = 252 |             a = ? |

    a = get_a();

This line does quite a lot. First, the frame register and return address are written to the stack, and space is reserved for the return value.

|  Address |           Variable |
| -------: | -----------------: |
| FR = 256 |  \<end of memory\> |
|      255 |          total = ? |
|      254 |              c = ? |
|      253 |              b = ? |
| SR = 252 |              a = ? |
|      251 |                256 |
|      250 |                  ? |
|      249 | \<return address\> |

Then the stack pointer and frame pointer are set to the top of the stack, and the program counter is set to the entry point of `<get_a()>`

|       Address |           Variable |
| ------------: | -----------------: |
|           256 |  \<end of memory\> |
|           255 |                  ? |
|           254 |                  ? |
|           253 |                  ? |
|           252 |                  ? |
|           251 |                256 |
|           250 |                  ? |
| SR = FR = 249 | \<return address\> |

    return 2;

There are no local variables, so this line writes `2` to `*(FR + 1)`, and sets the program counter to `*SR`.

|       Address |           Variable |
| ------------: | -----------------: |
|           256 |  \<end of memory\> |
|           255 |                  ? |
|           254 |                  ? |
|           253 |                  ? |
|           252 |                  ? |
|           251 |                256 |
|           250 |                  2 |
| SR = FR = 249 | \<return address\> |

    a = get_a();

Back to the caller:

+ The frame pointer is set to `*(FR + 2)`
+ The stack pointer is incremented by `3` (1 for the return address, 1 for the return value, and 1 for the frame pointer)

|  Address |          Variable |
| -------: | ----------------: |
| FR = 256 | \<end of memory\> |
|      255 |         total = ? |
|      254 |             c = ? |
|      253 |             b = ? |
| SR = 252 |             a = ? |
|      251 |               256 |
|      250 |                 2 |

Then the return value (found at `SR - 2`) is stored as "a" (found at `SR + 0`)

|  Address |          Variable |
| -------: | ----------------: |
| FR = 256 | \<end of memory\> |
|      255 |         total = ? |
|      254 |             c = ? |
|      253 |             b = ? |
| SR = 252 |             a = 2 |
|      251 |               256 |
|      250 |                 2 |

        b = get_b();
        c = get_c();

Similarly, `get_b()` and `get_c()` are called.

|  Address |          Variable |
| -------: | ----------------: |
| FR = 256 | \<end of memory\> |
|      255 |         total = ? |
|      254 |             c = 8 |
|      253 |             b = 5 |
| SR = 252 |             a = 2 |

Note how address 250 is used for several different values.

total = add_two_numbers(a, b);

This call is different, as there are paramers.

Before making the call, space is allocated for the parameters and return value, and the values are copied from local values.

|       Address |           Variable |
| ------------: | -----------------: |
|           256 |  \<end of memory\> |
|           255 |                  ? |
|           254 |                  ? |
|           253 |                  ? |
|           252 |                  ? |
|           251 |                256 |
|           250 |              b = 5 |
|           249 |              a = 2 |
|           248 |                  ? |
| SR = FR = 247 | \<return address\> |

    int c;

The stack pointer is decremented to make space for the local memory

|  Address |           Variable |
| -------: | -----------------: |
|      256 |  \<end of memory\> |
|      255 |                  ? |
|      254 |                  ? |
|      253 |                  ? |
|      252 |                  ? |
|      251 |                256 |
|      250 |              b = 5 |
|      249 |              a = 2 |
|      248 |                  ? |
| FR = 247 | \<return address\> |
| SR = 246 |                  ? |

    c = a + b;

|  Address |           Variable |
| -------: | -----------------: |
|      256 |  \<end of memory\> |
|      255 |                  ? |
|      254 |                  ? |
|      253 |                  ? |
|      252 |                  ? |
|      251 |                256 |
|      250 |              b = 5 |
|      249 |              a = 2 |
|      248 |                  ? |
| FR = 247 | \<return address\> |
| SR = 246 |                  7 |

    return c;

First, the value of "c" is copied to `*(FR + 1)`

|  Address |           Variable |
| -------: | -----------------: |
|      256 |  \<end of memory\> |
|      255 |                  ? |
|      254 |                  ? |
|      253 |                  ? |
|      252 |                  ? |
|      251 |                256 |
|      250 |              b = 5 |
|      249 |              a = 2 |
|      248 |                  7 |
| FR = 247 | \<return address\> |
| SR = 246 |                  7 |

Then, the value of SR is set to the value of FR, the program continues are per the earlier function call.

|  Address |          Variable |
| -------: | ----------------: |
| FR = 256 | \<end of memory\> |
|      255 |         total = 7 |
|      254 |             c = 8 |
|      253 |             b = 5 |
| SR = 252 |             a = 2 |

    total = add_two_numbers(total, c);

|  Address |          Variable |
| -------: | ----------------: |
| FR = 256 | \<end of memory\> |
|      255 |        total = 15 |
|      254 |             c = 8 |
|      253 |             b = 5 |
| SR = 252 |             a = 2 |

Note that:

+ `add_two_numbers` is called twice, with two different sets of paramers, but the same code was called, as the relevent values are copied to the "same" location relative to SR
+ All local variables are read from and written to addresses relative to either SR or FR. This allows for complex recursive function calls.

The stack allows for local variables, recursive functions, and much more complex code structures. However:

+ Returning a large number of values is expensive, as each value has to be copied from one stack frame to another
+ Returning a data structure with pointers to local values can easily lead to accessing memory that's not on the stack any more, or over-writing other values, leading to memory corruption bugs
+ Once an array has been allocated, it cannot be resized, thus leading to large buffers, "just in case"

There is thus a requirement for memory allocations which persist after the end of a stack frame.

The memory region where such memory blocks are allocated is called the heap

## The Heap

The heap is a region of memory that starts immediately after the Data Region, and finished when it overlaps with the stack. Thus a program can make efficient use of the entire memory space, even if it uses a lot of heap space and little stack space, and vice verse.

The memory layout is thus as follows

| End of memory |
| :-----------: |
|     Heap      |
|  \<Unused\>   |
|     Stack     |
|     Data      |

The heap is a *significantly* more complex data structure that the stack, and I'll gloss over the details on how and where blocks are allocated, how the cleanup works, etc, as there are several algorithms and methods of heap allocation that perform best is different circumstances.
I'll just talk about how the heap is used, and its strengths and shortcomings.

    int* get_list() {
        int* list;
        list = malloc(3);
        list[0] = 2;
        list[1] = 5;
        list[2] = 8;
        return list;
    }

    void print_list(int* list) {
        printf("0 => %d", list[0]);
        printf("1 => %d", list[1]);
        printf("2 => %d", list[2]);
    }

    void main() {
        int* list;
        list = get_list();
        print_list(list);
        free(list);
    }

At the start of `main`, the memory layout is as follows:

|       Address |          Variable |
| ------------: | ----------------: |
| SR = FR = 256 | \<end of memory\> |
|           255 |                 ? |
|           254 |                 ? |
|           ... |                 ? |

    int* list;

|  Address |          Variable |
| -------: | ----------------: |
| FR = 256 | \<end of memory\> |
| SR = 255 |          list = ? |
|      254 |                 ? |
|      ... |                 ? |

    list = get_list();

|  Address |           Variable |
| -------: | -----------------: |
| FR = 256 |  \<end of memory\> |
| SR = 255 |           list = ? |
|      254 |                256 |
|      253 |                  ? |
|      252 | \<return address\> |

Only one memory value is reserved, as the return value is a `int*` which is one memory value large.

|       Address |           Variable |
| ------------: | -----------------: |
|           256 |  \<end of memory\> |
|           255 |           list = ? |
|           254 |                256 |
|           253 |                  ? |
| SR = FR = 252 | \<return address\> |

    int* list;

|  Address |           Variable |
| -------: | -----------------: |
|      256 |  \<end of memory\> |
|      255 |           list = ? |
|      254 |                256 |
|      253 |                  ? |
| FR = 252 | \<return address\> |
| SR = 251 |           list = ? |

list is a single value, so one value of memory is reserved.

    list = malloc(3);

Malloc is a system call which reserves a section *somewhere* on the heap.
The library manages the heap, and I'll gloss over it.
For this demo, let's say that the heap starts at address 128.

|  Address |           Variable |
| -------: | -----------------: |
|      256 |  \<end of memory\> |
|      255 |                  ? |
|      254 |                256 |
|      253 |                  ? |
| FR = 252 | \<return address\> |
| SR = 251 |         list = 128 |
|      ... |                  ? |
|      130 |                  ? |
|      129 |                  ? |
|      128 |                  ? |

    list[0] = 2;
    list[1] = 5;
    list[2] = 8;

|  Address |           Variable |
| -------: | -----------------: |
|      256 |  \<end of memory\> |
|      255 |           list = ? |
|      254 |                256 |
|      253 |                  ? |
| FR = 252 | \<return address\> |
| SR = 251 |         list = 128 |
|      ... |                  ? |
|      130 |                  8 |
|      129 |                  5 |
|      128 |                  2 |

    return list;

|  Address |          Variable |
| -------: | ----------------: |
| FR = 256 | \<end of memory\> |
| SR = 255 |        list = 128 |
|      ... |                 ? |
|      130 |                 8 |
|      129 |                 5 |
|      128 |                 2 |

The function returns, the stack pointer and frame pointer and restored, and the return value is written to the address of list. Note that the heap allocation persists, and although the function "returns" an array of three items, only one memory value is copied.

    print_list(list);

Similarly, when making the call to `void print_list(int* list)`, only one memory value is copied.

    free(list);

|  Address |          Variable |
| -------: | ----------------: |
| FR = 256 | \<end of memory\> |
| SR = 255 |        list = 128 |
|      ... |                 ? |

The heap entry is deallocated. Note that `list` still points to address 128, but trying to dereference it could lead to a system crash or memory corruption. For safely, most programs will set a pointer to 0 after freeing it's heap block. However, that doesn't stop other pointer from pointing to its block...

### This machine vs C

Note that the forms of `malloc` and `free` are different than as per C, as this simplified system has fewer data types, as explained at the start of this document.

|       This system       |              C              |
| :---------------------: | :-------------------------: |
| `int* malloc(int size)` | `void* malloc(size_t size)` |
| `void free(int* start)` |  `void free(void* start)`   |

### Strengths of the Heap

+ Large data structures can be returned from functions without an expensive copying of memory; Only the pointer pointing to the memory needs to be copied
+ Arrays can be resized; A new block is allocated, the values copied, the old block freed, and the referencing pointer is replated with the address of the new block
+ Complex data structures such as linked lists can be instantiated and extended

### Drawbacks of the Heap

+ There's no automatic memory management; Unlike the stack, where values are automatically cleaned up at the end of the stack frame, heap entries must be explicitly freed by the program. If memory is not correctly managed, then the following errors might be introduced:
  + Memory leaks (a block of memory is `malloc`ed, but never `free`d)
  + Double free (`free` is called on the same block of memory twice)
  + Reference invalidation, such as a pointer to partway in an array that's been resized and thus freed

## Summary

+ The Data Segment is used for storing global variables
+ The Stack is used for storing local variables, function arguments, and return addresses (in some calling conventions)
+ The Heap is used for dyamic arrays, large object, and objects that should persist after the function returns
+ Stack frames are automatically "deallocated" by incrementing the stack pointer
+ Heap blocks must be manually freed
