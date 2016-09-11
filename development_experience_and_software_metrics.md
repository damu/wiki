# Development Experience and Software Metrics

## Development Experience

In software development experience is super important. Even if you went to some kind of school or studied
computer science / informatics / software development there's just so much they can't teach you and most teaches
are terrible software developers anyway. Gaining experience by actually doing things and trying out various
approaches is way more important.

Ideas for small projects:  
Small games. The common idea is to copy simple games like Pong, Tetris, Asteroids or Breakout. If one want to make game development
such games have the disadvantage of being rather special. Most games have some kind of moving character (platformer, shooter, ...). Sure
one can start with whatever but a kinda general small game project like a platformer (Mario) has more things that are likely to reappear
in later projects as well (player movement, enemies (AI), pick ups, ...).  
Small applications. Maybe some kind of file processing. Turning CSV data into JSON. Combining multiple images into one. Making a GUI
application with a slider and load & save functions to edit the brightness of an image. Processing of simple file formats like
Bitmap (.bmp), Wave (.wav) or Tar (.tar), the support for the file format doesn't have to be complete, it's more about exercising such
things in general and not about following every detail of the format. One could visualize a Wave file or show the content of a Tar archive
and allow file extraction, file adding and Tar archive creation. One could also implement AES to for example encrypt and decrypt a file with a password.

### Flagship Project

If one wants or needs to show his software development ability he should have one especially good project for this purpose. This project must be accessible in source form, at least for looking at it (license wise). It would be good if this project has a certain motivation behin it like: "I didn't like software/game X because of Y and Z so I made that part better/differently". The project should have a decent complexity and good quality in regard of software metrics:

## Software Metrics

There are several metrics of software development:
- performance (speed of execution)
  - absence of stutters (if the software sometimes or regularily freezes. This also includes a jumpy frame rate or frame rate drops in games)
- memory usage (does the software need an unnecessary amount of memory (including for very short times))
- software quality (stability and (user) error resistance)
  - maintainability and readability
- hardware usage and leakage (does the software create files scattered everwhere? Does it need "temporary" files and doesn't delete them?)
- dependencies (does the software require a giant amount of other software on the users computer or a certain configuration)
- annoying the user (some commercial software loves to annoy the user all the time with terrible default settings, pop-up messages and annoying behavior)
- development time (how fast one can get something done)

Performance is not only relevant for time critical applications like games or audio software. If you have a text editor that needs
10 seconds to start up or when loading a file and lags when scrolling or doing simple operations that's also terrible and a total no-go.
Today the common thinking is that performance doesn't matter at all because computers are so fast. My text editor (Libre Office) has
performance issues even when having only 10 pages of simple text. My browsers (Firefox, Chrome and Chromium) hang all the time when
loading a new page or when using "big pages" like Amazon or Youtube.  
I'm using multiple browsers because no browser can handle every site I want to access. Firefox can't handle Youtube performance wise.
Chromium can't play videos on Vimeo or Dailymotion for whatever reason.

When developing software one must take these metrics into account. There's so much terrible and annoying software out there. Most programs
have huge issues with several of these points.

By constant training and doing "training projects" one must strive for improvements in all areas. 
