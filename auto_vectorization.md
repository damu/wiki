# Auto Vectorization

Auto Vectorization is a compiler optimization that uses [SIMD](https://en.wikipedia.org/wiki/SIMD) instructions.
This is for example the SSE extension on Intel compatible CPUs.

SIMD often requires a special data layout, special data handling and is often done with hand written (SIMD) instructions, which basically means writing assembler. This is for once not portable and one does also not want to do this in a high language like C++. Most modern compilers can automatically create SIMD instructions if these can be used in the given case. GCC for example does them with -O3 or with the special auto vectorization flag.  
GCC does also have a flag that generates information output to see what he could optimize with SIMD and what he could not and why:
  -ftree-vectorizer-verbose=2
The number goes from 1 to 6 with 6 being the most verbose. In my test 2 seems to be the most useful one.

I started experimenting with various loops to see what is optimized and how good.

### Test 1: Simple Vector Math

```C++
std::vector<unsigned char> vec;   // make a vector with 10240*10240 elements
vec.resize(10240*10240);

for(auto& e:vec)                  // fill with random data
    e=rand();

for(auto& e:vec)                  // add 100 to every element
    e+=100;
```

Without auto vectorization (-O2) the "add 100 to every element" loop takes ~0.050 seconds.  
With auto vectorization (-O3) the "add 100 to every element" loop takes ~0.013935 seconds.  

By the way it didn't make any difference if I used such a range based for loop with `auto&`, one with `auto`, one with `auto&&`, a classical index based for loop or a pointer based loop.  
I tested all and there was no difference in speed. I had other cases where a pointer based loop was faster, so this can matter but didn't in this case. Maybe because of the memory being the main bottleneck.

The assembler output with auto vectorization contains:
```
movdqa xmm0,XMMWORD PTR [rdx+r11*1]
paddb  xmm0,xmm7
movdqa XMMWORD PTR [rdx+r11*1],xmm0
add    r11,0x10
```
These `xmm` registers are 128bit registers. In this case they contain 16 unsigned chars.  
The `movdqa` instructions moves data from memory to the xmm0 register and later from there back into memory.  
The `paddb` SIMD instruction adds the content of xmm7 (which is 100) to every one of the 16 chars.  
The index gets incremented by 16 (0x10) and the loop continues.

So instead of adding 100 to every single byte by itself this automatically SIMD optimized loop adds the 100 to 16 bytes at once.  
This optimization in this case makes the code "only" 4x faster and not 16x as one could assume. I guess it's limited by memory speed.

Let's see:  
It takes ~0.013935 seconds to process 104857600 bytes. That are ~7.525 GB per second.  
Without the SIMD optimization the speed was only 2.097 GB/s.  
The machine I tested this on is a virtual machine and the host has [DDR2 RAM](https://en.wikipedia.org/wiki/DDR2_SDRAM). I don't know what exact type of RAM it is  and how much other processes and the virtual machine cost drain but Wikipedia says the fastest
DDR2 has a transfer rate of 8.533 GB/s. The 7.525 GB/s that I measured are surprisingly close and that was reading, calculating and writing.

I didn't use any special instruction or any kind of alignment. The compiler automatically created code that works also with unaligned data by using non-SIMD code until it reaches properly aligned data.  
There are special alignment commands and special keywords. They may have made this slightly faster, but also more complicated. This basically worked without nowing anything about SIMD.
