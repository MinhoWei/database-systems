# Cost Models
Cost

**Time cost**: total time taken to execute a method (we do not consider this)

**Page cost**: # of pages read / written

**Page** := fixed-size block of data -> the size is determined by storage medium

Terminologies:

r: # of tuples in a relation

b: # of pages

B: size of page

c: max # of tuples in a page

The tuple which answer query q are contained in b<sub>q</sub> pages

T<sub>r</sub>: page read time (high)

T<sub>w</sub>: page write time (high)
