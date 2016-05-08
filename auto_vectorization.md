# Auto Vectorization

Auto Vectorization is a compiler optimization that uses [SIMD](https://en.wikipedia.org/wiki/SIMD) instructions.
This is for example the SSE extension on Intel compatible CPUs.

SIMD often requires a special data layout, special data handling and is often done with hand written (SIMD) instructions, which basically means writing assembler. This is for once not portable and one does also not want to do this in a high language like C++. Most modern compilers can automatically create SIMD instructions if these can be used in the given case. GCC for example does them with -O3 or with the special auto vectorization flag.  
GCC does also have a flag that generates information output to see what he could optimize with SIMD and what he could not and why:  
`-ftree-vectorizer-verbose=2`  
The number goes from 1 to 6 with 6 being the most verbose. In my test 2 seems to be the most useful one.  
The output is quite chaotic but contains something like
```
..\main.cpp:8: note: not vectorized: not enough data-refs in basic block.
Vectorizing loop at ..\main.cpp:28
..\main.cpp:14: note: not vectorized: number of iterations cannot be computed.
..\main.cpp:14: note: bad loop form.
..\main.cpp:8: note: vectorized 1 loops in function.
```

I started experimenting with various loops to see what is optimized and how good.

## Test 1: Simple Vector Math

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
The machine I tested this on is a virtual machine and the host has [DDR2 RAM](https://en.wikipedia.org/wiki/DDR2_SDRAM). I don't know what exact type of RAM it is and how much other processes and the virtual machine drain but Wikipedia says the fastest
DDR2 has a transfer rate of 8.533 GB/s. The 7.525 GB/s that I measured are surprisingly close and that was reading, calculating and writing.

I didn't use any special instruction or any kind of alignment. The compiler automatically created code that works also with unaligned data by using non-SIMD code until it reaches properly aligned data.  
There are special alignment commands and special keywords. They may have made this slightly faster, but also more complicated. This basically worked without nowing anything about SIMD.

The same test with a vector of `unsigned short` instead of `unsigned char` did take 0.027901 seconds. The assembler code had `paddw`
instead of `paddb` which adds the 100 to every word (2 bytes) in the 128bit register (8 shorts instead of 16 chars).  
Here the speed was 7.516 GB/s (104857600 shorts are 209715200 bytes). The speed is practically the same, the measured times are always fluctuating a bit which explains the small difference.


## Test 2: Minimum Maximum

I looked at a SSE2 instruction list and saw instructions that compare and assign in one step. I didn't even know that there are such operations in general. They are called conditional moves.

### First Try

Same test code as above but different loop action:
```C++
std::vector<unsigned char> vec;
...
for(auto& e:vec)
    e=std::min<unsigned char>(e,200);
```
So I want to ceil every element in a vector to 200 (so that every element is <=200).  
My normal approach is to use std::min (or std::max).

This took 0.093441 seconds. Compared to the measured times in the first test this does not look good SIMD wise.  
I looked at the generated assembler output and there was no SIMD there.  
But it contained at least `cmova` which is a "conditional move if above".

The vectorizer info output says
```
Analyzing loop at ..\main.cpp:28
..\main.cpp:28: note: versioning for alias required: can't determine dependence between MEM[(const unsigned char &)SR.375_338] and *SR.375_338
..\main.cpp:28: note: misalign = 0 bytes of ref D.103094
..\main.cpp:28: note: misalign = 0 bytes of ref D.103094
..\main.cpp:28: note: not vectorized: complicated access pattern.
..\main.cpp:28: note: bad data access.
...
..\main.cpp:8: note: vectorized 0 loops in function.
```

The std::min functions looks surprisingly simple though:
```C++
template<typename _Tp>
inline const _Tp&
min(const _Tp& __a, const _Tp& __b)
{
    // concept requirements
    __glibcxx_function_requires(_LessThanComparableConcept<_Tp>)
    //return __b < __a ? __b : __a;
    if (__b < __a)
        return __b;
    return __a;
}
```
The `__glibcxx_function_requires` is just a compile time thing and does not generate actual code. So the function should be
identical to:
```C++
inline const unsigned char& min(const unsigned char& __a, const unsigned char& __b)
{
    if (__b < __a)
        return __b;
    return __a;
}
```
Such a function should get completely inlined.  
No idea why the ternary operator is commented out in my STL (same in MinGW 5.3.0 and 4.8) and why they use two return instead. Maybe that causes the not-auto-vectorization?

### Second Try

The second version of this test was:
```C++
for(auto& e:vec)
    e=e>200?200:e;
```
This one takes only 0.016846 seconds. That are 6.224 GB/s.

The auto vectorizer info says:
```
Analyzing loop at ..\main.cpp:28
..\main.cpp:28: note: Unknown misalignment, is_packed = 0
..\main.cpp:28: note: Unknown misalignment, is_packed = 0
..\main.cpp:28: note: virtual phi. skip.
..\main.cpp:28: note: not ssa-name.
..\main.cpp:28: note: use not simple.
..\main.cpp:28: note: not ssa-name.
..\main.cpp:28: note: use not simple.
..\main.cpp:28: note: virtual phi. skip.
Vectorizing loop at ..\main.cpp:28
...
..\main.cpp:8: note: vectorized 1 loops in function.
```

The generates assembler contains SIMD this time:
```
movdqa xmm0,XMMWORD PTR [rax+r11*1]
pminub xmm0,xmm7
movdqa XMMWORD PTR [rax+r11*1],xmm0
add    r11,0x10
```
`pminub` is a conditional SIMD move that compares 16 bytes at once.

## Test 3: Getting Complexer

```C++
for(size_t i=0;i<vec.size();i+=4)
{
    vec[i]+=50;
    vec[i+1]+=100;
    vec[i+2]+=150;
    vec[i+3]+=200;
}
```
This approach with operator[] gets not vectorized in any way and takes 0.0615 seconds.

A pointer based approach instead:
```C++
unsigned char* p_end=vec.data()+vec.size();
for(unsigned char* p=vec.data();p<p_end;p+=4)
{
    p[0]+=50;
    p[1]+=100;
    p[2]+=150;
    p[3]+=200;
}
```
This takes 0.014694 seconds and gets vectorized into:
```
movdqu xmm0,XMMWORD PTR [r9]
add    r11,0x1
add    r9,0x10
paddb  xmm0,xmm7
movdqu XMMWORD PTR [r9-0x10],xmm0
cmp    r11,r10
jb     0x40304f <main(int, char**)+367>
```
When using directly `vec.data()+vec.size()` instead of `p_end` in the `for` the loop gets not vectorized and takes 0.038044 seconds.
