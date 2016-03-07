# Library Design: Creating Objects Handled by the Library

For a while I exclusively used variadic templates to let the library create object:
```C++
widget->create_child<lfgui::button>("click me");
```
Inside the library the button is managed via a unique_ptr.

This approach had the major disadvantage of having no help by the IDE at all (as in showing the parameters / possible overloads) and of having way to complicated compiler errors.

Then I added the approach done by Qt and many others:

```C++
widget->add_child(new lfgui::button("click me"));
```

Now there is help by the IDE and good compiler errors if parameters are wrong.  
BUT: There's a `new` in the code which looks wrong.  
The passed object is internally still handled with a unique_ptr but `new` looks like an error and as if the user has to handle the ressource.  
Also I really hate this creation with `new` in Qt.

The other possibility I can think of is:
```C++
widget->add_child(std::make_unique<lfgui::button>("click me"));
```
- It's longer.  
- `make_unique` is not in C++11.
- There is no IDE help.
- Compiler errors are again somewhere in the library as with my first approach..

It would be also possible to pass by value or reference and copy the object, but in this case the object can't be copied.

...  
Oh I could try to use r-value reference (T&&) so that this may work:

```C++
widget->add_child(lfgui::button("click me"));
```

That I would like.

Tried that. Program crashed with "Unknown Signal received". wat. Something could be wrong with my move constructors. Maybe I'll try that later again. Should be possible in principle but seems to be Black Magic (an experimental and new field where compiler don't really help that much).  

