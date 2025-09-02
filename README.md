# You already know C++

C is bad. The best and worst part of C++ is it's compatability with C. 

This means we inherit the best parts of C and the worst.

However C++ is good and offers the high level, safe alternatives we've come to expect from languages.
I'll show you what these are, where to find them and how C++ is simpler than you probably gave it credit for.

---

## C is evil

In C the programmer is required to keep track of all heap storage, i.e.:
- Before usage size must be allocated
- This size cannot change
- Must be freed when done with
- Cannot be used again after free

Modern languages handle all this for us, C++ offers all common data structures without need to mess with memory.

---

## How to be safe

Use the standard library.

Most common problems people have can be solved by using the standard library:
- Buffer overflow? Use `std::vector` and `std::string`.
- Use after free? Use a vector `std::unique_ptr` or `std::shared_ptr`.
- Access out of bounds? Use bound checked lookups like `.at(size_t)`
- Iterating through a container? use `for (auto item : container)`

The vast majority of unsafe code does things in a C style rather than a C++ style. For example it is unlikely that a file handle would be leaked with C++'s `std::ostream` and much more likely with C's `fopen`.

---

## Still struggling? WWPD

**W**hat **W**ould **P**ython **D**o?

I think python and C++ are very similar and I think good C++ should resemble python.

For example our python code for creating a list of size 10 containing 0 (ints):
```python
import ctypes
size_t = ctypes.c_size_t
int32 = ctypes.c_int32
size_of_list = 10
size_of_list_bytes = (32/8) * size_of_list
libc = ctypes.CDLL("libc_malloc_debug.so")
data_ptr = libc.malloc(size_of_list_bytes)
```
No? you don't write python like that? then why write C++ like that?

- I need an array A with 10 int inside?

`A :list[int] = [0] * 10;`
`std::array<int, 10> A{0};`

- Okay but I need it to be resizable.
`A :list[int] = [0] * 10;`
`std::vector<int> A(10, 0);`

- Cool now how do I iterate through it?
`for a in A: code(a)`
`for (auto a : A) { code(a); }`

- How do I delete the memory so I don't leak it?
Python: ` ` (Garbage collector handles it)
C++: ` ` (Item leaves the stack and destructor handles it)

---

## I can't use the standard library

If you are given a char pointer (`char*`) rather than a string (also known as a CString) and not something like `std::string` you should get it into something "known safe" as soon as possible. This means you'll need to check the length of it.

In C the length of a string is from the first character to the first string termination character. So to check the length you have to loop through it.

A common issue occurs when using: `size_t strlen(const char* str)` which will happily go forever and read out of bounds if string termination character is not found.

If the length of the buffer (where the string is held) is known we can use that to limit how far we look for the null terminating character with `size_t strnlen_s(const char* str, size_t max_length)`.

Now we know the string is safe and we have the length of the string we can copy it into something less combersome to deal with like a `string` or a `string_view` (assuming we know the lifetime of the object).

## But UB?

If you've done something wrong the compiler is likely to tell you about it.

To be told about more possible things you've done wrong enable more compiler warnings.

For example this code compiles with no warnings on gcc but when ran, crashes because of an infinite loop.
```C++
template <typename T>
struct C {
    void foo() { static_cast<T*>(this)->foo(); }
};

struct D : C<D> { };

void f(D *d) { d->foo(); }

int main() {
    D d{};
    f(&d);
}
```
We can let gcc know that actually we maybe want to be warned about such sillyness with the flag `-Winfinite-recursion` or another one that includes it such as `-Wall`.

This gives us the output: 
```
<source>: In member function 'void C<T>::foo() [with T = D]':
<source>:3:10: warning: infinite recursion detected [-Winfinite-recursion]
    3 |     void foo() { static_cast<T*>(this)->foo(); }
      |          ^~~
<source>:3:44: note: recursive call
    3 |     void foo() { static_cast<T*>(this)->foo(); }
      |                  ~~~~~~~~~~~~~~~~~~~~~~~~~~^~
```

To force collegues to listen to these use: `-Werror` or if the code is already filled with warnings turn some into errors with `-Werror=` and hide it in the build script so they can't turn it off.

---

## But UB!

UB is short for undefined behaviour, this is something that the C++ standard (usually intentionally and explictly) has not specified, usually because it would force a compramise onto the user. 

For example if I add two numbers and attempt to store the result in a space that is too small to store that number I can:
1. Store the result in a larger amount of memory, check if it's bigger than max of the small amount of memory, crash or put it in the smaller bit of memory.
2. Use some CPU trickery to set a flag if an overflow has occured
3. Do all maths with dynamically sized storeage and grow it if required
4. Do a saturated add
5. Do an overflow

In 1 we slow the code down (a little, mostly through register contention), in 2 we limit compilation to CPUs with this special magic, in 3 we limit ourselves to processors with dynamically resizable memory (no GPUs), 4 and 5 make enforcements on the processor that might benfit one but be terrible for another (cannot mask int maths as float maths on certain GPUs)


You can run your code with UB sanitizers, this means that any code that executes UB will cause a crash. In gcc it's two lines.

[//]: # (Vertical slide)

### But Rust!
For this example Rust crashes on debug but overflows on Release, it's effectively the same as C++ but with slower int maths on some GPUs (compromise 5).

---

## BUT UB!!

Still not convinced?

The `constexpr` keyword was introduced in C++14 and runs code at compile time*. Any code ran at compile time cannot execute undefined behaviour, so will result in a compilation failure if attempted. So any code you can test at compile time is guarenteed to not contain undefined behaviour.

[//]: # (Vertical slide)

\* Technically constexpr code must run any time before the program runs, so on embeded devices this might be on device start but realisticlly it's at compile time. 

Again the standard is written this way to allow for different processors to handle things differently, for example if I am compiling for some processor that has much more precise float handling, I probably don't want to use the processor I am compiling the code with to do all my maths.

---

## But garbage collection?

## Functions

----

### GC: Functions:
We know that any object going into a function as an argument has a lifetime greater than the local scope of the function.

```C++
void foo(int& in) {
	// We know that in is always valid in this scope
	in += 1;
}

// We know that a is always valid in this scope
int a = 1;
foo(a);
```
Here `a` goes through `foo` as argument `in`, we know the whole time it's safe.


[//]: # (Vertical slide)

Following the above statement, we (and the compiler) know that any local data inside the function we want to end up outside of the local scope must be moved (or copied) to that outer scope. So:
```C++
int foo(int& in) {
	int i = 1;
	return i; // (free) Implicit move  
}
```
Or a copy if object is not movable
```
struct NoMove {  NoMove(NoMove&&) = delete; };
NoMove foo() {
	return NoMove{}; // Implicit copy (cannot move because we deleted it), (also usually free)
}
```

[//]: # (Vertical slide)

If we do something silly like: `int& foo() { return 1; }` the compiler errors. If we are more determined about our sillyness, `int& foo() { int i = 0; return i}`, the compiler warns us:
```
warning: reference to local variable 'i' returned [-Wreturn-local-addr]
   10 | int& foo() { int i = 0; return i; }`
	  |                                ^
```
So naturally, use `-Werror=return-local-addr`. (Enforced in C++26)

---

### GC: Classes:



We can normally see structs or classes (the same thing) as a very similar extenstion of this concept: 
```C++
struct IntRefWrapper {
	IntRefWrapper(int& wrappedInt) : m_wrappedInt{wrappedInt} {}
	int& m_wrappedInt;
};

{
	int a = 1;
	IntRefWrapper b{a};
}
```










