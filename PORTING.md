# Porting guide

This document will guide you through the process of getting this toolchain up and running on an unsupported firmware. The only implicit assumption made here is that the core exploit (exploit.js) does run on the target firmware. It shouldn't be too difficult to swap that for another userland exploit, but it's not discussed here.

This is more-or-less what I did for 6.51 (I didn't have any userspace dumps back then). 6.72 happened to be "compatible enough", so no more porting was required.

# 1. Find GOT offset

Dump WebKit using `python3 bad_hoist/memserver/dump_module.py -1` (this should never fail). Then disassemble the dumped binary, extract all `CALL imm32` instructions, and sort by the target address:

`objdump -D bad_hoist/dumps/webkit.elf | grep 'call[^%]*$' | cut -d "$(printf '\t')" -f 3- | sort | uniq | less`

You'll see something like this in the dump:

```
callq  ffffffffff0b9c41 <__bss_start+0xfffffffffd7aec41>
callq  ffffffffff2bf176 <__bss_start+0xfffffffffd9b4176>
callq  ffffffffff65df97 <__bss_start+0xfffffffffdd52f97>
callq  ffffffffff667308 <__bss_start+0xfffffffffdd5c308> <-- probably the start of the GOT
callq  ffffffffff667318 <__bss_start+0xfffffffffdd5c318>
callq  ffffffffff667338 <__bss_start+0xfffffffffdd5c338>
callq  ffffffffff667348 <__bss_start+0xfffffffffdd5c348>
callq  ffffffffff667358 <__bss_start+0xfffffffffdd5c358>
callq  ffffffffff667368 <__bss_start+0xfffffffffdd5c368>
callq  ffffffffff667378 <__bss_start+0xfffffffffdd5c378>
callq  ffffffffff667388 <__bss_start+0xfffffffffdd5c388>
callq  ffffffffff667408 <__bss_start+0xfffffffffdd5c408>
callq  ffffffffff667418 <__bss_start+0xfffffffffdd5c418>
```

Here `0xffffffffff667308` is the first of a few hundreds of similarly looking entries. Probably it's the start of the GOT (or PLT, don't care much about the terminology). This means that the GOT is located `(1 << 64) - 0xffffffffff667308` bytes below the leaked virtual method.

Now insert this offset into `bad_hoist/dumpers/dump_got.js` at line 11, and load a page that loads exploit.js, helpers.js, malloc.js, dumpers/dump_got.js. `print` function should be somehow routed to your dev PC.

If the script warns about "decrement!", this means that there is one more GOT entry that didn't get referenced. Increment the offset by 16.

# 2. Find GOT entries corresponding to the required modules

Run the dump_got.js script from above, this time with the correct offset. You'll get a long list of pointers, some of which are `0xeffffffe00000000`, and some others are actual code pointers. Save this output to a file (it will be useful later) and sort the pointers by their value.

Here for each module are a few dozens of pointers pointing into it. If two consecutive pointers differ by hundreds of megabytes, then most probably the previous module ended here and a new one just started. Example:

```
80bfc5820
80bfc59c0
<-- here
812a5b5f0
812a83970
```

In a 6.72 dump, 5 modules can be obviously separated this way. It turns out that both libc and libkernel are amongst them.

# 3. Dump modules

Once you know a "base address" (not a base address actually, but rather address of some random function inside the module) you can try dumping the module. Go to the file you saved at step 2 (a fresh dump won't be useful because of userland ASLR) and search for the (copy-pasted from sorted) lowest pointer of a specific module. If the pointer is at line X+1, then this module can be dumped by running `python3 bad_hoist/memserver/dump_module.py X`.

(Note: before dumping, replace the GOT offset in dump_module.py with the offset from dump_got.js)

Now dump each of these modules and check if it's either libc or libkernel:

* `libc` is the module that contains a pivot() gadget (`mov rsp, [rdi+0x38] ; pop rdi ; ret`). The gadget is located at the end of an easily recognizable loadall() function.
* `libkernel` contains easily recognizable syscall wrappers. Search for `syscall` in the disassembly; if what you see looks like a syscall wrapper, you've just found libkernel.

Note: as the base address used is not the real base address, some important function may be located below that address. To address this case, `dump_module.py` supports a `index,offset` notation as its argument (see `bad_hoist/memserver/Makefile`). E.g. to get `pthread_create` into the dump I had to pass `705,-0x10000` instead of just `705`, so that dumping starts 64KB lower than the pointer.

Once you've verified that you'd dumped the right modules, fix `bad_hoist/memserver/Makefile` with the correct offsets and run the dumping scripts in "production mode":

```
bad_hoist/ $ make clean
bad_hoist/ $ make
```

# 4. Fix hardcoded offsets in `rop/rop.js`

1. Line 16. Insert the correct GOT offset.
2. Lines 33-34. Insert the correct libc & libkernel bases. The numbers are the same as in Makefile.
3. Lines 35-43:

3.1. `loadall` is the function in libc that contains the pivot() gadget.

3.2. `saveall` is a twin function of `loadall` that saves registers instead of restoring them.

3.3. `setjmp` and `longjmp` are not used, as `loadall` and `saveall` play the same role here. You can just strip these out.

3.4. `pivot` is the pivot() gadget, see above.

3.5. `infloop` is a EB FE (`.loop\njmp loop`) gadget, not used in the end exploit but is useful for debugging ropchains.

3.6. `jop_frame` is a JOP gadget that reads as `push rbp ; mov rbp, rsp ; mov rax, [rdi] ; call qword [rax]`. It is used for cleanly returning from subthreads, see `ps4-rop-8cc/librop/extcall.{c,h}`.

3.7. `get_errno_addr` is a function that returns the address of the errno(3) variable for the current thread. To find this address, follow the `jb` jump after any syscall instruction, then follow an indirect jump, and you will see a CALL to this function.

3.8. `pthread_create` is the only function in libkernel that calls the `thr_new` syscall wrapper. First find the wrapper by searching for the thr_new syscall number, then find a call to this wrapper, then search backward for a function prologue.