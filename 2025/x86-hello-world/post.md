Yes, you can still run this snippet:

```c
const int main[] = {
    -443987883, 440, 113408, -1922629632,
    4149, 899584, 84869120, 15544,
    266023168, 1818576901, 1461743468, 1684828783,
    -1017312735
};
```

but you'll need to do some tweaks.

After compiling without the GCC nonsense using `gcc code.c -nostartfiles -nostdlib -e main -o ./code`,
we can clearly see that the code is placed in `.rodata`:

```
❯ objdump -s ./code

./code:     file format elf64-x86-64

Contents of section .note.gnu.build-id:
 400158 04000000 14000000 03000000 474e5500  ............GNU.
 400168 2c2f3d22 7e1307a2 fbe75478 c3f440ce  ,/="~.....Tx..@.
 400178 939745b8                             ..E.            
Contents of section .note.gnu.property:
 401000 04000000 20000000 05000000 474e5500  .... .......GNU.
 401010 010001c0 04000000 01000000 00000000  ................
 401020 020001c0 04000000 00000000 00000000  ................
Contents of section .rodata:
 401040 554889e5 b8010000 00bb0100 0000678d  UH............g.
 401050 35100000 00ba0d00 00000f05 b83c0000  5............<..
 401060 0031db0f 0548656c 6c6f2057 6f726c64  .1...Hello World
 401070 210a5dc3                             !.].            
Contents of section .comment:
 0000 4743433a 2028474e 55292031 352e322e  GCC: (GNU) 15.2.
 0010 31203230 32353038 30382028 52656420  1 20250808 (Red 
 0020 48617420 31352e32 2e312d31 2900      Hat 15.2.1-1).  
```

which is quite logical for compile-time `const`s.

Now, since `.rodata` ends up in a read-only and non-executable memory page,
we can't really run this without some tuning. 🏎️

This security policy happens inside `execve`, so the easiest workaround is to run it in an insecure `execve` :-)

```
❯ uvx ulexecve ./code
Hello World!
```

or...

tell the compiler to write to a different section.

```c
__attribute__((section (".text"))) const int main[] = {
    -443987883, 440, 113408, -1922629632,
    4149, 899584, 84869120, 15544,
    266023168, 1818576901, 1461743468, 1684828783,
    -1017312735
};
```

Crazy stuff.

It will compile with this cute little warning:

```bash
❯ gcc different-section.c -nostartfiles -nostdlib -e main -o ./code
/tmp/cc3gU6zF.s: Assembler messages:
/tmp/cc3gU6zF.s:4: Warning: ignoring changed section attributes for .text
```

And... it worked!

```bash
❯ objdump -s ./code

./code:     file format elf64-x86-64

Contents of section .note.gnu.build-id:
 400190 04000000 14000000 03000000 474e5500  ............GNU.
 4001a0 448b40f6 12e260a3 de7465ac 6da141ee  D.@...`..te.m.A.
 4001b0 2a2cd530                             *,.0            
Contents of section .text:
 4001c0 554889e5 b8010000 00bb0100 0000678d  UH............g.
 4001d0 35100000 00ba0d00 00000f05 b83c0000  5............<..
 4001e0 0031db0f 0548656c 6c6f2057 6f726c64  .1...Hello World
 4001f0 210a5dc3                             !.].            
Contents of section .note.gnu.property:
 401000 04000000 20000000 05000000 474e5500  .... .......GNU.
 401010 010001c0 04000000 01000000 00000000  ................
 401020 020001c0 04000000 00000000 00000000  ................
Contents of section .comment:
 0000 4743433a 2028474e 55292031 352e322e  GCC: (GNU) 15.2.
 0010 31203230 32353038 30382028 52656420  1 20250808 (Red 
 0020 48617420 31352e32 2e312d31 2900      Hat 15.2.1-1).  
```

```bash
❯ ./code
Hello World!

❯ strace ./code
execve("./code", ["./code"], 0x7ffe15527610 /* 110 vars */) = 0
write(0, "Hello World!\n", 13Hello World!
)          = 13
exit(0)                                 = ?
+++ exited with 0 +++
```

:-)
