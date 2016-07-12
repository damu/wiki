# Return Value Optimization

I was trying to optimize an image scaling function.

(The measured times here may sound unsignificant small but it was only a test with a small image and normal ones are often
40 times as big. Also depending on the case even these short times may be way to long.)

I surprisingly found out that the whole function takes twice as long as the scaling itself and measured the single steps:
```C++
image scaled_bilinear(int w,int h) const
{
    if(width()<=0||height()<=0) // return empty image if this one is empty
        return image(w,h);

    image ret(w,h);
// 0.006088 seconds until this point (allocation and initialization of the image)

... // scaling
// 0.017158 seconds for the scaling itself

    return ret;
}
```
The whole function took 0.038867 seconds for my test with a small image.  
The remaining ~0.015 seconds are lost when returning the image. It seems to be doing a full copy.

I changed the code a bit to:
```C++
image image::scaled(int w,int h) const
{
    image ret(w>0?w:1,h>0?h:1);
    if(width()<=0||height()<=0)
        return ret;

... // scaling
// again around 0.017 seconds for the scaling itself

    return ret;
}
```
Having only one image defined and that also as the first thing causes the compiler to move the allocation and initialization
out of this function and does completely save the ~0.015 seconds when returning the image (there's no copy being made at all).  
This function takes only 0.017 seconds. Though it practically also still takes the ~0.006 seconds for creating the image.

The most surprising thing for me was that returning the image (one copy I guess?) takes so long compared to the
complicated scaling that involves way more math and memory jumping.

### std::vector sucks

Later I also optimized the image to no longer use a std::vector to hold the pixel data. A std::vector has the bad property of
always initializing every element in the vector. Which is pretty unwanted when overwriting the values shortly after anyway
like in this case. Here this unneeded initialization took ~0.006 seconds.  
Compared to the second version here this replacement of std::vector with an own container made upscaling an image in one test
nearly twice as fast (from ~0.040 to ~0.025 seconds).  
There's no way to tell a std::vector to not initialize it's elements. There's nothing like resize_without_init(X). I consider
that an STL bug.
