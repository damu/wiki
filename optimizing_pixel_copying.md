# Optimizing Pixel Copying

So I have an image from a library I'm using and want to copy the pixels in that image to another image in a different format.
The first image has the color channels separated, first the red values of all pixel, then blue, then green and then alpha. The second image has the pixels in the more common pixel based format with: First pixel: red, green, blue, alpha; next pixel: red, green, blue, alpha; next pixel: ...
Both images have all pixel data in a row without any spaces and both images have the same size (width and height) and the same color depth and channel count (both RGBA with 8bit per red, green, blue and alpha).

The first simple approach was:

for(int y=0;y<height();y++)        // go through each coordinate
    for(int x=0;x<width();x++)     // having y in the outer and x in the outer loop has already cache optimization reasons due to the memory layout.
    {
        // get each pixel from the first image with separated channels with the access functions that returns a color struct with {uint8_t r,g,b,a;}
        lfgui::color p=this->img.get_pixel(x,y);
        // convert this color to a color of the second library
        _image->SetPixel(x,y,Urho3D::Color(p.r/255.0f,p.g/255.0f,p.b/255.0f,p.a/255.0f));
    }

This code took 0.047 seconds. The application was a real-time visualization (like a game) and just this code brought my FPS (frames per second) down from ~190 to ~18, which is an unacceptable lag.

The first thing that jumps to the mind are the floating point operations (expensive!) at the color conversion. Luckily the second library does also have a more direct SetPixelInt(uint32_t) function:

for(int y=0;y<height();y++)        // go through each coordinate
    for(int x=0;x<width();x++)
    {
        // get each pixel from the first image with separated channels with the access functions that returns a color struct with {uint8_t r,g,b,a;}
        lfgui::color p=this->img.get_pixel(x,y);
        // the lfgui::color has also a uint32_t with RGBA (via a union) so just set those 32bit directly
        _image->SetPixelInt(x,y,p.value);
    }

This code took 0.031 seconds and brought my application to 24 FPS.

Another thing that I saw was the usage of the width() and height() functions in the loop. The compiler may not optimize those away so let's do it manually and see:

int h=height();
int w=width();
for(int y=0;y<h;y++)        // go through each coordinate
    for(int x=0;x<w;x++)
    {
        // get each pixel from the first image with separated channels with the access functions that returns a color struct with {uint8_t r,g,b,a;}
        lfgui::color p=this->img.get_pixel(x,y);
        // the lfgui::color has also a uint32_t with RGBA (via a union) so just set those 32bit directly
        _image->SetPixelInt(x,y,p.value);
    }

This code took 0.03 seconds and no visible FPS change. A minor optimization but it's faster.

The next thing I saw was that the pixel can also be set directly in memory. The SetPixelInt does have to calculate the pixel position in the memory by doing calculations with the x and y coordinate (it may also do additional stuff like checks). This time can be saved:

int h=height();
int w=width();
// get a pointer to the pixel data
uint32_t* data_target=(uint32_t*)_image->GetData();
for(int y=0;y<h;y++)
    for(int x=0;x<w;x++)
    {
        lfgui::color p=this->img.get_pixel(x,y);
        *data_target=p.value;        // copy the 32bit directly to memory (without coordinate calculations)
        data_target++;               // increment pointer (it's automatically incremented by 4 as the thing it's pointing too is 4 bytes big)
    }

Now it takes 0.022 seconds and I have 33 FPS.

The still existing get_pixel function can also be optimized as it is also doing coordinate calculations that are not necessary:

int h=height();
int w=width();
int count=width()*height();
int countx2=count*2;
int countx3=count*3;
// due to the channel separation in the first image, the pixels have to be copied byte/color vise
uint8_t* data_target=_image->GetData();
uint8_t* data_source=this->img.data();
uint8_t* data_source_end=this->img.data()+count;
for(;data_source<data_source_end;)
{
    *data_target=*data_source;
    data_target++;
    *data_target=*(data_source+count);
    data_target++;
    *data_target=*(data_source+countx2);
    data_target++;
    *data_target=*(data_source+countx3);
    data_target++;
    data_source++;
}

Wow this takes only around 0.001-0.002 seconds and I'm at ~90 FPS! (my timer is getting imprecise)
I got rid of all multiplications, unnecessary operations and object creations. Now only pointer increments, additions and byte movements (the CPU loves that).
