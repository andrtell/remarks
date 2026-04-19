# 2. Dynamic linking

## 2.1 A tiny dynamically linked library

Lets start by writing the source code for our example library __easymath__.

`./easymath.h`

```c
#ifndef EASYMATH_H
#define EASYMATH_H

int easymath_add(int a, int b);

#endif
```

`./easymath.c`

```c
int easymath_add(int a, int b) {
	return a + b;
}
```

### Creating a shared object file

We are going to attempt to use our library by dynamically linking to it.

To do that we need a shared object file `libeasymath.so`.

To compile the shared object file `libeasymath.so` for our library we can use the __`-fPic`__ and __`-shared`__ compiler flags.

```
> cc -fPIC -shared -o libeasymath.so easymath.c
```

Ok, lets see what we have so far.

```
> ls

easymath.c
easymath.h
libeasymath.so*  # <- this is new.

> file libeasymath.so

libeasymath.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked,
BuildID[sha1]=39fec9a5cb0e0b6d41f4407918906e17ba18b192, not stripped
```

Great! We have our shared object file `libeasymath.so`.

### Using the library in a program

Lets write a program that makes use of the function `easymath_add` in our library.

`./program.c`

```c
#include <stdio.h>
#include <easymath.h>

int main(void) {
  int v = easymath_add(1, 2);
  printf("%d + %d = %d\n", 1, 2, v);
}
```

Lets try and compile the executable `program` for our program.

```
> cc program.c -o program

program.c:2:10: fatal error: easymath.h: No such file or directory
    2 | #include <easymath.h>
      |          ^~~~~~~~~~~~
compilation terminated.
```

Oh no! An error. The compiler complains it can not find the header file `easymath.h`.

### Locating the header files

We can fix the previous error by using the __`-I`__ flag to specifiy the directory of the header file `easymath.h`.

```
> export LIB_DIR=$(pwd)  # <- this is new.

> cc -I $LIB_DIR \       # <- this is new.
	 program.c   \
	 -o program

/usr/bin/ld: /tmp/ccCtxbLO.o: in function `main':
program.c:(.text+0x13): undefined reference to `easymath_add'
collect2: error: ld returned 1 exit status
```

Dang! Another error. This time the linker complains it can't find the reference `easymath_add` in `main`.

### Linking with our library

The linker needs to know which shared object file provides `easymath_add` so it can record the dependency in the final executable `program`.

To make this happen we first use the __`-l`__ flag to tell the compiler to link our executable `program` 
with the shared object file `libeasymath.so`.

We also need to use the __`-L`__ to specify the directory of the shared object file `libeasymath.so`.

```
> export LIB_DIR=$(pwd)

> cc -I $LIB_DIR \
   	 -L $LIB_DIR \  # <- this is new.
	 program.c   \
	 -leasymath  \  # <- this is new. Note: -leasymath = lib + easymath + .so
	 -o program
```

Phew! We managed to compile our executable `program`.

```
> ls

easymath.c
easymath.h
libeasymath.so*
program*  # <- this is new.
program.c

> ldd program

linux-vdso.so.1 (0x00007f6d6e5ae000)
libeasymath.so => not found
libc.so.6 => /usr/lib/libc.so.6 (0x00007f6d6e200000)
/lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007f6d6e5b0000)
```

Above `ldd` was used to print a list of the shared object files required by the executable `program`.

Note that `libeasymath.so` was listed, but that its path resolves to: `not found`.

Ok, lets try and run our executable `program`.

```
> ./program

./program: error while loading shared libraries: libeasymath.so: cannot open shared object file: No such file or directory
```

Shoot! A third error. 

### Providing the runtime path of our library

When our executable `program` runs it is up to the dynamic linker (`ld.so`) to _find_ and load any shared object files
required by our program. I seems like the dynamic linker needs some help here.

One way to do this is to set the environment variable `LD_LIBRARY_PATH` to the directory of our shared object
file `libeasymath.so` before running our program. 

```
> LD_LIBRARY_PATH=$(pwd) ./program

1 + 2 = 3
```

A second way to do this is to embedd the _runtime_ directory of the shared object file `libeasymath.so` in the executable
`program`. We can do this by using the __`Wl,-rpath,`__ flag during compilation.

```
> export LIB_DIR=$(pwd)

> cc -I $LIB_DIR \
   	 -L $LIB_DIR \
   	 program.c   \
	 -leasymath  \
   	 -Wl,-rpath,$LIB_DIR \  # <- this is new.
   	 -o program
```

Fantastic! Lets check our executable `program` with `ldd` again. 

```
> ldd ./program

linux-vdso.so.1 (0x00007ffa39d55000)
libeasymath.so => /some/dir/libeasymath.so (0x00007ffa39d43000)
libc.so.6 => /usr/lib/libc.so.6 (0x00007ffa39a00000)
/lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007ffa39d57000)
```

Ok, looks good. Lets try it out.

```
> ./program

1 + 2 = 3
```

Success!

## References

_man 1 gcc_

```
-shared

  Produce  a  shared  object  which  can  then  be  linked  with  other objects
  to form an executable.

  Not all systems support this option.

  For predictable results, you must also specify the same set of options used for
  compilation (-fpic, -fPIC, or model suboptions)  when  you  specify this linker option.[1]


-fPIC

  If supported for the target machine, emit position-independent code,
  suitable for dynamic linking and avoiding any limit on  the  size  of  the
  global offset table.

  This option makes a difference on AArch64, m68k, PowerPC and SPARC.

  Position-independent code requires special support, and therefore works
  only on certain machines.

  When this flag is set, the macros "__pic__" and "__PIC__" are defined to 2.

-I dir

    Add the directory dir to the list of directories to be searched for header files during preprocessing...

-Ldir

    Add directory dir to the list of directories to be searched for -l.

-Wl,option

    Pass option as an option to the linker.
    If option contains commas, it is split into multiple options at the commas.
    You can use  this  syntax to  pass an argument to the option.
    For example, -Wl,-Map,output.map passes -Map output.map to the linker.
    When using the GNU linker, you can also get the same effect with -Wl,-Map=output.map.
    NOTE: In Ubuntu 8.10 and later versions, for LDFLAGS, the option -Wl,-z,relro is used.
    To disable, use -Wl,-z,norelro.
```

_man 1 ld_

```
-rpath=dir

    Add a directory to the runtime library search path.

    This is used when linking an ELF executable with shared objects.

    All -rpath arguments are concatenated and passed to the runtime linker,
    which uses them to locate shared objects at runtime.

    The  -rpath  option  is  also  used  when  locating  shared objects which
    are needed by shared objects explicitly included in the link

    [...]

    If -rpath is not used when linking an ELF executable, the contents of
    the environment variable "LD_RUN_PATH" will be used if it is defined.
```

