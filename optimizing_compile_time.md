# Optimizing Compile Time

## Typical Tips for Reducing Compilation Time 

Some tips often found on the internet, I may have forgotten big ones:

- avoid a lot of code
- avoid big header files, move code into source files
- only include files that are actually needed
- avoid templates (especially nested and recursive ones)
  - templates can be hidden inside files not included everywhere so that only specific instantiations/usages are exposed in widely used files
- pre-compiled header files
- (unity builds, not helpful when usually only partly builds are needed)

## Optimization Try Example

The compile time of [LFGUI](https://github.com/LFGUI/LFGUI) started getting annoying and I tried reducing it. The IDE I used (QtCreator) displays the time a build process needs in whole seconds, which I used to measure the time.

I moved various longer function from header files into source files. Functions defined in classes are automatically inline which has usually only advantages for short functions, so moving definitions out of the classes has no disadvantage in that perspective.  
The project doesn't have that many source files (compilation units). One thing that I noticed which surprise was that adding a new source file by moving long functions from a header file did actually increase compilation time. I started at 15 seconds and got 16 seconds with one small new source file (in my specific case, the time depends a lot on the case).

After testing various things I ended up with only creating one new source file, several moved functions from header files into existing (and the new) source file and one big external header file being no longer included in a header but in a source file (the new one).  
My compilation time for a full build was still at 15 seconds. But doing just a part rebuild when editing something feels way faster (haven't measured that). Reason is logically that the header files are much lighter now.

The most time is actually spend on the cIMG library: a giant single header library with 53588 lines in that single header file!   
One of the first things I did way before trying to reduce compilation time again was moving this included header from a header file into a source file. The compilation time before was way to high. This header file alone takes ~10 seconds to compile and that for each compilation unit (source file) it is somehow included in!
