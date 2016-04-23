# strict std::function

### The Problem

std::function behaves differently in the libstdc++ (GCC) and in the Visual Studio 15 STL which caused a project of mine to not
work in Visual Studio 15.  
I don't know which of the two libraries or compilers is wrong but I found a fix to the following problem:
```C++
#include <iostream>
#include <functional>

// two functions which take different std::function's (lambdas) to act differently
void foo(std::function<void()> f)    {std::cout<<"void"<<std::endl;}
void foo(std::function<void(bool)> f){std::cout<<"bool"<<std::endl;}

int main()
{
    foo([]{});          // call directly with a lambda
    foo([](bool){});
    
    std::function<void()> f([]{})
    std::function<void(bool)> f2([](bool){});
    foo(f);             // call with a lambda wrapped in a std::function
    foo(f2);
}
```
This code works with GCC/MinGW but not with Visual Studio:
```
error C2668: 'foo': ambiguous call to overloaded function
note: could be 'void foo(std::function<Return (bool)>)'
note: or       'void foo(std::function<Return (void)>)'
```

I need this to work as I want to overload a function with different std::function's. I need the functions to act differently
depending on which parameters the lambda has.

.  

This following code does also do interestingly different stuff:
```C++
std::function<void()>([](bool){});
```
I'm trying to give a lambda with one parameter (a bool) to a std::function that expects a lambda with no parameters.

GCC complains:
```
...main.cpp:194:37: error: no matching function for call to 'std::function<void()>::function(qMain(int, char**)::__lambda44)'
note: candidates are:
...
```
As expected/hoped it tells one a proper error at exactly that line that caused it.

Visual Studio on the other hand:
```
c:\program files (x86)\microsoft visual studio 14.0\vc\include\type_traits(1434): error C2672: 'std::invoke': no matching overloaded function found
c:\program files (x86)\microsoft visual studio 14.0\vc\include\functional(208): note: see reference to function template instantiation 'void std::_Invoke_ret<_Rx,_Callable&>(std::_Forced<_Rx,true>,_Callable &)' being compiled
...
```
You can see the complete error message here: https://justpaste.it/tjvy

Visual Studios whole error doesn't say at all which line caused the error. It's all full of Visual Studio STL internal stuff.

### The Solution

To fix this error I had to make a function class that works like the std::function in libstdc++ (GCC). After many hours of
searching and testing I found a working solution that works with C++11. C++17 will have a std::is_callable that may have made
this much easier but I couldn't find a proper implementation of that so I kinda came up with my own.

```C++
template<typename Function,typename Return,typename... Parameter>
class is_callable
{
public:
    static const bool value=std::is_same<decltype(&Function::operator()),Return(Function::*)(Parameter...)const>::value;
};

template<typename Return=void,typename... Parameter>
class function_strict
{
    std::function<Return(Parameter...)> func;
public:
    function_strict(){}

    template<typename T,typename = typename std::enable_if<is_callable<T,Return,Parameter...>::value>::type>
    function_strict(T func):func(func){}

    Return operator()(Parameter... parameter)const{return func(parameter...);}
};
```
The first class checks if a given lambda has the given return type and the given parameters. If this is the case, its
data member `value` will have the value `true`, otherwise `false`.

It can be used standalone like:
```C++
auto func=[](bool){};
cout<<is_callable<decltype(func),void,bool>::value<<endl;
```

The second class `function_strict` uses this first class to check if a given lambda matches the expected signature to provide
a "stricter" std::function as the Visual Studio STL.  
This function_strict does work with GCC and Visual Studio:
```C++
// two functions which take different lambdas to act differently
void bar(function_strict<> f)         {std::cout<<"void"<<std::endl;}
void bar(function_strict<void,bool> f){std::cout<<"bool"<<std::endl;}

int main()
{
    bar([]{});          // call directly with a lambda
    bar([](bool){});

    function_strict<void> f([]{});
    function_strict<void,bool> f2([](bool){});
    bar(f);             // call with a lambda wrapped in a std::function
    bar(f2);
}
```

The syntax is slightly different compared to std::function as here the template parameters are not given
as `std::function<void(bool)>` but as `function_strict<void,bool>`.  
I couldn't figure out how to do that, if that is even possible.

.  

For some reason GCC doess also accept passing two std::function's to the `bar` function which takes a `function_strict`:
```C++
    std::function<void()> f([]{});
    std::function<void(bool)> f2([](bool){});
    bar(f);             // call with a lambda wrapped in a std::function
    bar(f2);
```
Visual Studio does not accept this due to the wrong type.

.    

PS: The C++ template syntax is terrible.  
Also I had hoped to never have to use member function pointers again.
