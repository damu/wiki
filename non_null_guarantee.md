# Non Null Guarantee

In one of his videos Casey Muratori (https://www.youtube.com/watch?v=QauD5cAgnT8) ranted about references and mentioned that they can be 0 like a pointer and claimed that they are totally useless due to that and other reasons. It was one of his silly C++ rants fueled by his insane hate and several wrong claims.  
I dug into that "reference with 0" and did some research:  
A somewhat common misconception is that referencees can't be 0. They can. That can be found out pretty quickly when searching for it as well, like here: http://stackoverflow.com/questions/57483/what-are-the-differences-between-a-pointer-variable-and-a-reference-variable-in  
But dereferencing 0 is actually Undefined Behaviour: http://en.cppreference.com/w/cpp/language/ub  
The problem is: the compiler (at least GCC up to 6.2.0 as it seems) doesn't care at all. There is no way to get a warning for that kind of Undefined Behavior and it works just fine (kinda, it crashes of course when using the reference). I hate that there is no "warn me about all cases of Undefined Behavior in my code"-option  
There is a warning option to detect dereferencing 0 but that one doesn't trigger most of the time. It's called   
-Wnull-dereference and requires -fdelete-null-pointer-checks: https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html

Examples:

```C++
#include <iostream>
#include <string>
#include <memory>
using namespace std;

struct person
{
  string name="Peter";
};

void do_stuff(const person& p)
{
  cout<<"do_stuff "<<p.name<<endl;
}

int main()
{
  do_stuff(*(person*)0);              // does not trigger a warning
  {
    person* p=0;
    cout<<"do_stuff "<<p->name<<endl; // does not trigger a warning
    do_stuff(*p);                     // does not trigger a warning
  }
  {
    shared_ptr<person> p=nullptr;
    cout<<"do_stuff "<<p->name<<endl; // does not trigger a warning
    do_stuff(*p);                     // does not trigger a warning
  }
  return 0;
}
```
So having a function taking a reference does not prevent a null pointer issue/crash at all.

I found out that some compilers have a nonnull attribute to set that a function is not allowed to take a null pointer, so let's try that:
```C++
#include <iostream>
#include <string>
#include <memory>
using namespace std;

struct person
{
    string name="Peter";
};

void do_stuff(person* p) __attribute__((nonnull (1)));
void do_stuff(person* p)
{
  cout<<"do_stuff "<<p->name<<endl;
}

int main()
{
  do_stuff((person*)0);                 // triggers actually a compiler warning
  {
    person* p=0;
    do_stuff(p);                        // does not trigger a warning
  }
  {
    shared_ptr<person> p=nullptr;
    do_stuff(p.get());                  // does not trigger a warning
  }
  return 0;
}
```
This is not really better.  
By the way the warning looks like this:
```
..\pointer_test\main.cpp: In function 'int main()':
..\pointer_test\main.cpp:19:22: warning: null argument where non-null required (argument 1) [-Wnonnull]
   do_stuff((person*)0);                 // triggers actually a compiler warning
                      ^
```

I compiled both of these with GCC 6.2.0 and these options:
```
-O2 -Wall -Wpedantic -Wextra -Wnull-dereference -fdelete-null-pointer-checks -Wuninitialized -pedantic-errors
```

And I also compiled both of these with GCC 4.8.2 with these options. I had to remove -Wnull-dereference since it is too new:
```
-O2 -Wall -Wpedantic -Wextra -fdelete-null-pointer-checks -Wuninitialized -pedantic-errors
```

Maybe there are other (external) tools or other compilers that can detect such "mistakes".  

This sucks.  
Such more extensive statical analysis when compiling is one of the main ideas behind the Rust programming language.

Is there a way to detect such simple mistakes at compile time? Not really it seems. Would be nice if the incoming concepts would allow "custom" compile time checks like that.  
The only thing I came up with is a runtime solution:

## Runtime Solution

One could use a class/struct that checks for 0 when the pointer or references gets set/changed. A reference in C++ is pretty much a const pointer (a pointer that can't be set to anything else). The same can be done with a wrapping class/struct. This could be similar to std::unique_ptr.

One could also check for 0 whenever dereferencing the class/struct, expecially if 0 is valid but when dereferencing 0 should actually be caught. This would be expensive though, so maybe only do this in debug mode (like with "#ifdef DEBUG").  
  
##   
  
I think I never had a "0 reference" bug, but I had a bunch of pointers being 0 when not expecting that and bugs due to that (in software already shipped to buying customers as well, by the way). Most of the time I don't check for 0 somewhere when I know that it can never be 0 at that time. But it happened a few times that I later change stuff and make that object optional so that it can actually be 0 at that time and forget to add checks -> bug causing a crash  
Such bugs can be really hard to find. A runtime check (in debug mode) when using a 0 pointer/reference would have saved a lot of time.
