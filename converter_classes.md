# Converter Classes

This is a bit similar to the article "[Better Typed C++](better_typed_c++.md): Issues with C++ types and conversions and fixing them with current C++".

### The Problem

One wants to write functions that take an object pointer in various fashions, like T*, unique_ptr<T>, shared_ptr<T>, ...
But one does not want to write every single overload.

Also one does not want to add an implicit T* conversion to the wrapping classes (like some unique_ptr and shared_ptr).

### Possible Solution

One can write a converter class that has the overloads for every desired conversion. With this approach the number of
overloads gets moved from every function wanting to accept various pointers to the converter class.

```C++
#include <iostream>
using namespace std;

// some example class
class person
{
public:
    person()    {cout<<"person() "<<(void*)this<<endl;}
    ~person()   {cout<<"~person() "<<(void*)this<<endl;}
};

// some class wrapping a pointer (this is not a proper unique_ptr as there are functions missing!)
template<typename T>
class unique_ptr
{
    T* ptr;
public:
    explicit unique_ptr(T* ptr) : ptr(ptr) {}
    ~unique_ptr(){delete ptr;}
    T* get() const {return ptr;}
};

// the converter class. It accepts T* and unique_ptr<T>.
template<typename T>
class any_pointer
{
    T* ptr;
public:
    any_pointer(T* ptr) : ptr(ptr) {}
    any_pointer(const unique_ptr<T>& ptr) : ptr(ptr.get()) {}
    operator T*() {return ptr;}
};

void pointer_nacked(person*)                    {cout<<"pointer_nacked(person*) called"<<endl;}
void pointer_unique(const unique_ptr<person>&)  {cout<<"pointer_unique(unique_ptr<person>) called"<<endl;}
void pointer_any(any_pointer<person>)           {cout<<"pointer_any(any_pointer<person>) called"<<endl;}

int main()
{
    person* nacked=new person;
    pointer_nacked(nacked);
    //pointer_unique(nacked);   // error: could not convert 'nacked' from 'person*' to 'unique_ptr<person>'

    unique_ptr<person> unique(new person);
    //pointer_nacked(unique);   // error: cannot convert 'unique_ptr<person>' to 'person*' for argument '1'
                                // to 'void pointer_nacked(person*)'
    pointer_unique(unique);

    // the pointer_any function accepts person* and unique_ptr<person> due to the converter class
    pointer_any(nacked);  
    pointer_any(unique);

    delete nacked;
    return 0;
}
```

Output:
```
person() 0x735c80
pointer_nacked(person*) called
person() 0x735ca0
pointer_unique(unique_ptr<person>) called
pointer_any(any_pointer<person>) called
pointer_any(any_pointer<person>) called
~person() 0x735c80
~person() 0x735ca0
```
