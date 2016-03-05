# Better Typed C++

I had code like this:
```C++
void foobar(long);
void foobar(float);
void foobar(bool);
void foobar(std::string);
...
foobar("hello, world");
```
Which function is called?  
One would clearly expect the one with std::string since "hello, world" is a text. NOPE! It's the one with bool. There's not even a warning (GCC 5.4.1).

Also when using
```C++
foobar(42);      // int
foobar(13.37);   // double
```
GCC doesn't know which function to use and aborts with an error. One would expect he would here pick the most likely one. The long one for the int 42 and the float one for the double 13.37.

I have no idea who made these weird rules. The ambiguous overload could be explained by saying "yeah it's a strict language" but when why the fuck does he call a bool (!) when having a "hello, world"?? A text string like "hello, world" has the type char*. A bool is also saved in one byte like a char, one could see that as the reason but that is clearly not intended. Also:
```C++
foobar('A');      // char
```
This one is also ambiguous. With a char* he picks bool without even a warning but a char at the same time is not accepted as a bool??  
  
  
So I starting thinking about what could be done to fix this.  
In the case where I encountered these errors the foobar was a constructor so just not using the same name for the function was not really an option. I used enable_if to resolve the issue which ended up looking like this:
```C++
// This one takes every integer, but no bool's.
template<typename T,typename std::enable_if<std::is_integral<T>::value&&
                                           !std::is_same<T,bool>::value,bool>::type=true>
bar(T integer);

// This one takes every floating point.
template<typename T,typename std::enable_if<std::is_floating_point<T>::value,bool>::type=true>
bar(T floating_point);

// This one takes only bool.
template<typename T,typename std::enable_if<std::is_same<T,bool>::value,bool>::type=true>
bar(T boolean);

// For std::string.
bar(const std::string& string);

// For char*, like "hello, world".
bar(char* string);
```
What the $!#& is that $!#&? That syntax is way to complicated to use and I still don't exactly know why it works.
It's nice to have such options in C++ but that syntax...  
Maybe concepts or static-if improve this situation a bit, somewhere in the future...  
  
  
So I starting thinking about what could be done to fix this now.  
The problem is some conversions happen without being intended and others not when intended.

I started with really strict types:
```C++
template<typename T>
struct explicit_type
{
    T value;

    // had to use enable_if magic to get the strictness/explicitness I wanted
    template<typename T2,typename std::enable_if<std::is_same<T,T2>::value,bool>::type=true>
    explicit_type(T2 value):value(value){}

    explicit_type(const explicit_type<T>&)=default;
    explicit_type(explicit_type<T>&&)=default;
    explicit_type& operator=(const explicit_type<T>&)=default;
    explicit_type& operator=(explicit_type<T>&&)=default;
    operator T(){return value;}
};

using t_bool=explicit_type<bool>;
using t_char=explicit_type<char>;
using t_uchar=explicit_type<unsigned char>;
using t_short=explicit_type<short>;
using t_ushort=explicit_type<unsigned short>;
using t_int=explicit_type<int>;
using t_uint=explicit_type<unsigned int>;
using t_long=explicit_type<long>;
using t_ulong=explicit_type<unsigned long>;
using t_longlong=explicit_type<long long>;
using t_ulonglong=explicit_type<unsigned long long>;
using t_i8=explicit_type<int8_t>;
using t_i16=explicit_type<int16_t>;
using t_i32=explicit_type<int32_t>;
using t_i64=explicit_type<int64_t>;
using t_u8=explicit_type<uint8_t>;
using t_u16=explicit_type<uint16_t>;
using t_u32=explicit_type<uint32_t>;
using t_u64=explicit_type<uint64_t>;
using t_float=explicit_type<float>;
using t_double=explicit_type<double>;
using t_std_string=explicit_type<std::string>;
...
void bar(t_long v)         {cout<<"int"<<endl;}
void bar(t_float v)        {cout<<"float"<<endl;}
void bar(t_bool v)         {cout<<"bool"<<endl;}
void bar(t_std_string v)   {cout<<"string"<<endl;}
...
//bar(42);                         // int: doesn't work because no overload takes an int
bar(42l);                          // long: picks the t_long one as a t_long can be constructed from a long
//bar(13.37);                      // double: doesn't work because no overload takes a double
bar(13.37f);                       // float: -> t_float
bar(true);                         // bool: -> t_bool
//foo('A');                        // char: doesn't work because no overload takes a char
//bar("hello, world");             // char*: doesn't work because no overload takes a char*
bar(std::string("hello, world"));  // std::string: -> std::string
```
But I also wanted my original functions to work:
```C++
void foobar(long);         // this one should take all integral numbers
void foobar(float);        // this one all floating point numbers
void foobar(bool);         // only bool
void foobar(std::string);  // all kinds of text
```
So I started with some more general types that can be created out of various types:
```C++
// takes all kinds of integers (signed and unsigned) but not bool. (Takes also char which may not be intended
// and could be avoided like bool is avoided.)
struct t_integral
{
    long value;

    template<typename T2,typename std::enable_if<std::is_integral<T2>::value&&
                                                !std::is_same<T2,bool>::value,bool>::type=true>
    t_integral(T2 value):value(value){}

    t_integral(const t_integral&)=default;
    t_integral(t_integral&&)=default;
    t_integral& operator=(const t_integral&)=default;
    t_integral& operator=(t_integral&&)=default;
    operator long(){return value;}
};

// takes all kinds of floating point number
struct t_floating_point
{
    double value;

    template<typename T2,typename std::enable_if<std::is_floating_point<T2>::value,bool>::type=true>
    t_floating_point(T2 value):value(value){}

    t_floating_point(const t_floating_point&)=default;
    t_floating_point(t_floating_point&&)=default;
    t_floating_point& operator=(const t_floating_point&)=default;
    t_floating_point& operator=(t_floating_point&&)=default;
    operator double(){return value;}
};

// accepts std::string and char* like "hello, world"
struct t_string
{
    std::string value;

    t_string(const std::string& value):value(value){}
    t_string(const char* value):value(value){}

    t_string(const t_string&)=default;
    t_string(t_string&&)=default;
    t_string& operator=(const t_string&)=default;
    t_string& operator=(t_string&&)=default;
    operator std::string(){return value;}
};
...
void bar(t_integral v)       {cout<<"integral"<<endl;}
void bar(t_floating_point v) {cout<<"floating point"<<endl;}
void bar(t_bool v)           {cout<<"bool"<<endl;}
void bar(t_string v)         {cout<<"string"<<endl;}
...
bar(42);                           // integral
bar(42l);                          // integral
bar(uint8_t(42));                  // integral
bar('A');                          // integral
bar(13.37);                        // floating point
bar(13.37f);                       // floating point
bar(true);                         // bool
bar("hello, world");               // string
bar(std::string("hello, world"));  // string
```
So one can get very explicit types and also "sane implicit conversions" by making custom types with some C++11 magic. This can be moved in a library so that the non-expert-user doesn't have to bother and it's also simply reusable without having to lookup the weird enable_if syntax each time.
