# Data Structure Implementation in C, C++, and Python: Abstraction, Cost, and the Foundations of Machine Learning Systems

The first week of the course I wrote a stack in Python. Six lines, four of them class boilerplate. `push` was `self.items.append(x)` and `pop` was `self.items.pop()`. It worked on the first run, which annoyed me slightly, because I had expected to learn something and instead had typed out two method calls.

Then I wrote the same stack in C. Sixty lines: a `struct` with a data pointer and integers for size and capacity, a doubling routine on `realloc`, and a segmentation fault the first time I pushed past the initial eight slots, because I had updated `capacity` before checking whether `realloc` returned a valid pointer. I spent about forty minutes on that fault. It taught me more than the Python version had.

Which is roughly the point of a course that insists on all three languages. The syllabus is one set of ideas seen from three distances, and how much you can see of the machinery depends on where you are standing.

## The same abstraction, three memory models

A dynamic array is the cleanest example, since all three languages have one and they are doing the same thing underneath.

In C you own the memory. You call `malloc` for an initial block, you track how many slots are used and how many exist, and when they are equal you `realloc` to something larger, usually double, because growing by a constant amount turns a sequence of *n* appends into O(n²) copying. Then you free it, exactly once, on every path out of the function including the error paths. Nothing checks your indices. Writing to `arr[capacity]` is undefined behaviour, which in practice means it works fine on your machine for a week and then corrupts an unrelated variable during the demo.

In C++ the same structure is `std::vector`. It grows on `push_back` with the same doubling logic (libstdc++ and libc++ use a factor of 2, MSVC 1.5), templates let one implementation serve `vector<int>` and `vector<Node*>` alike, and the destructor releases the buffer at scope exit. The manual work is still there. Somebody else did it once, correctly.

In Python the structure is `list`, and it is simply there. CPython over-allocates on append, growing the buffer by roughly an eighth of the current size plus a small constant, which is why `append` is amortised O(1) even though one append occasionally copies the whole array.

## What the abstraction hides, and why that matters

Python does not make data structures easy so much as invisible, and the difference shows up as soon as complexity starts to matter.

In C, an expensive operation is expensive in a loop you wrote and can look at. In Python the cost sits behind a method call that looks identical to a cheap one. `lst.append(x)` and `lst.insert(0, x)` are the same length on screen; one is amortised constant time and the other shifts every element right by one position and is O(n). Put the second inside a loop and you have quietly written an O(n²) algorithm that looks like an O(n) one. The fix is `collections.deque`, which gives O(1) insertion at both ends, and it is an easy fix, but only if you knew there was a problem.

The same trap catches membership tests. `if x in my_list` scans linearly; `if x in my_set` hashes and checks a bucket in average constant time. On a thousand elements nobody notices. On a hundred thousand, inside another loop, it is the gap between a program that finishes and one you kill after ten minutes. I lost an afternoon to exactly this on a duplicate-detection problem before I thought to check what `in` costs on a list.

Python removes the implementation burden and leaves the analysis burden intact. If anything the analysis gets harder, because the cost is no longer visible in your own code.

## Where C is the better teacher

Linked structures are where I stopped resenting C. You can write a linked list in Python. A `Node` class with a `next` attribute, objects assigned to it, short and readable, and it teaches you almost nothing about what a link is, because Python hands you a reference and never lets you look at it. In C, `node->next` is an address. Deleting a node from the middle means holding a pointer to the previous node, or holding a pointer to the pointer that points at the current node, which only makes sense once you accept that the links themselves have locations.

The absence of safety nets does real teaching work. An off-by-one in Python raises `IndexError` with a line number; the same mistake in C corrupts the heap silently and shows up somewhere unrelated. That is why I now write down a loop's invariant before writing the loop.

## Where C++ closes the loop

C++ is where the two halves meet, and the standard library is the reason. After hand-writing a hash table with linear probing and then a binary search tree, the difference between `std::map` and `std::unordered_map` stopped being trivia to memorise. `std::map` is a balanced binary search tree, a red-black tree in every mainstream implementation, so its operations are O(log n) and iteration comes out in sorted key order. `std::unordered_map` is a hash table: average O(1) lookup, worst case O(n) under bad collisions, no ordering guarantee. Before I built both by hand they were two containers that did approximately the same job, and now the names tell me what I am getting.

I am not going to ship my own red-black tree. The point of writing one was that the library stopped being opaque.

## The layer beneath machine learning frameworks

This stopped being academic when I started reading how numerical libraries are built, because the split runs along the same line. A Python list of a million floats is an array of pointers to a million separately allocated objects, scattered across the heap. A NumPy `ndarray` holding the same values is one contiguous block of raw doubles, a C array with a shape attached. That is why a vectorised NumPy operation can beat the equivalent Python loop by one to two orders of magnitude: no per-element object overhead, and the cache gets to do its job.

The pattern repeats a level up. A PyTorch tensor operation is a thin Python call dispatching into compiled C++ kernels, so writing `torch.matmul` is the six-line stack again, correct and fast and completely opaque unless someone has written the version underneath.

Even model code that never leaves Python runs into this. A k-nearest-neighbours classifier scans every training point unless a spatial structure sits behind it, which is why scikit-learn exposes `algorithm='kd_tree'` and `algorithm='ball_tree'` next to brute force. Picking between those three is a data structures decision wearing a machine learning label.

## What transfers, and what does not

The syntax does not transfer. I still look up whether it is `len(x)` or `x.size()` or `x.length` every single time.

The reasoning does. A breadth-first search is a queue, a visited set, and a loop that pops one node and pushes its unvisited neighbours, in all three languages. The queue is `std::queue`, or `collections.deque`, or a struct with head and tail indices you wrote yourself; the visited set is `std::unordered_set`, or `set`, or an array of flags indexed by node ID. Only the containers change.

One limitation is worth recording, because nobody mentioned it in the videos: CPython's default recursion limit is 1000 frames. A depth-first search on a long path graph raises `RecursionError` at input sizes where the C++ version runs fine. Raising the limit with `sys.setrecursionlimit` trades a clean exception for a possible interpreter crash. The real fix is to rewrite the recursion with an explicit stack, which is the same conversion the C version would have forced on me anyway.

## How I work now

The routine I have settled into came from watching how long my mistakes took to find. The problem gets solved on paper first: what the state is, what the loop invariant is, and what complexity I am aiming for. Then a prototype in Python, because when the logic is wrong I want a stack trace in two seconds rather than a segfault in twenty minutes. Then, if the structure itself is the point of the exercise, a reimplementation in C or C++ and a comparison against the standard-library version.

The step I skipped longest was measuring, because knowing a bound is not the same as knowing which of two O(n log n) implementations is faster on real input. Six lines in Python and sixty in C do the same job. Knowing why the other fifty-four exist is what the course is for.
