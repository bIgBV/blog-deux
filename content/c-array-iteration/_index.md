+++
title = "Iterating over an array in c"
date = 2017-10-04

[taxonomies]
categories = ["programming", "c"]
+++

In the first semester of grad school, I wanted to revisit my basics. Therefore,
I took the [Operating Systems](http://cs385.class.uic.edu/) class. We take an
existing teaching operating system (MIT's xv6) and implement features like
memory mapping and threads on top. Having only sparingly worked with C and
never with assembly, I knew I was in for quite a ride.

As a part of the course, we were required to write a simple graphics driver.
The driver itself was pretty simple, where we wrote data to a memory region and
sent control commads on a pre-defined port. To test the driver we had to
display an image on the screen. So as things go with writing programs it was
2 in the morning and me questioning my computer "Why aren't you printing the
god damned image!". But of course, the computer was doing exactly what I asked
it to.

To write the file to the memory buffer, I had a loop which was iterating over
the contents of the file and writing it to the memory location. Sounds simple
enough right, this is what it looked like:

```c
int* i;

for(i = 0; i < n; i++) // n is the number of bytes to write
{
    display[(i + f->off)] = buf[i]; // display is a pointer to the region in memory
}
```

And this worked for the most part, but the image displayed was zoomed in and
alternate pixles were not coloured in. After a lot of time spent in gdb unable
to understand what was going on, I asked what was going on the class'
discussion forum. The professor pointed out that when we have an index of
a given type, when incrementing the index, we increment by the `sizeof` the
index variable. So in my loop, instead of iterating over each byte in the file,
I was iterating by `sizeof(int)`.

Once I used the correct type, it just worked.

So to conclude. When using an index to iterate over an array, everytime you
increment the index, you are actually incrementing by the `sizeof(type)`.
