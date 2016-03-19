I wanted to optimize my LFGUI code by not drawing the GUI when it is not visible. I started by clearing the Urho texture every frame if the GUI's root is invisible and noticed that the Image::Clear function was several times slower as my more complicated pixel copying which is done every frame when the GUI is visible.  
The Clear function sets every pixel in the image to a given value.

This is the code used in Clear:
```C++
unsigned char* src = (unsigned char*)&uintColor;
    for (unsigned i = 0; i < width_ * height_ * depth_ * components_; ++i)
        data_[i] = src[i % components_];
```
This takes 14.853 ms.

First try:
```C++
if(components_==4)
{
    uint32_t* data=(uint32_t*)data_.Get();
    for (unsigned i = 0; i < width_ * height_ * depth_; i++)
        *(data+i) = uintColor;
}
else
{
    unsigned char* src = (unsigned char*)&uintColor;
    for (unsigned i = 0; i < width_ * height_ * depth_ * components_; ++i)
        data_[i] = src[i % components_];
}
```
This copies all four bytes at once when a pixel consists of four bytes (ARGB).  
This takes only 0.678 ms.

Final optimization:
```C++
if(components_==4)
{
    uint32_t color=uintColor;
    uint32_t* data=(uint32_t*)GetData();
    uint32_t* data_end=(uint32_t*)(GetData()+width_ * height_ * depth_ * components_);
    for (; data < data_end; data++)
        *data = color;
}
else
{
    unsigned char* src = (unsigned char*)&uintColor;
    for (unsigned i = 0; i < width_ * height_ * depth_ * components_; ++i)
        data_[i] = src[i % components_];
}
```
This takes only 0.382 ms.

This optimizations has been proposed: https://github.com/urho3d/Urho3D/pull/1262
