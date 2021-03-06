# 7. Memory manipulation
These are little details that are usually several levels below the usual level of abstraction, but they can be handy from time to time. Here are a few use cases:
* Writing or reading binary data
* Some part is performance critical (and optimisation can be easier than parallelism)
* Reinventing the wheel (if the usual one doesn't fit your needs)
* Cheating around the standard

These rules apply to usual architectures like amd64, x86 or ARM. Some unusual architectures may behave differently. Also, compilers tend to have compiler-specific settings that can cause nonstandard memory layouts (such as `#pragma pack`).

---
## Quick question
How would you implement a generic container that can have multiple implementations?

---
## Data in struct or class
A structure fundamentally differs from a primitive type:
* A primitive type that isn't larger than the word size (which is almost always true on 64-bit architectures) will be saved on an address divisible by its size or with offset divisible by its size
* A structure will be saved on an address so that each of the primitive types it contains or its members contain will be saved on an address divisible by its size (can be checked using `alignof`)

Together, it means that a structure containing `int32_t`, `uint16_t`, `int16_t` and `uint8_t` will be on an address divisible by 4 because the largest variable's size is 4 bytes.

While it is usually possible for a variable to exist on an adress that isn't divisible by its size, its usage is inefficient and thus memory manipulation tricks or nonstandard compiler intrinsics are needed to place them there.

---
The offsets of members can be exactly determined
```C++
struct A {
	int a1; // offset 0
	int a2; // offset 4
	bool a3; // offset 8
	uint16_t a4; // offset 10
	double a5; // offset 16
	std::array<uint16_t, 3> a6; // offset 24
	std::unique_ptr<A> a7; // offset 32
	bool a8; // offset 40
};
struct B : A {
	int b1; // offset 44
};
```
To play with this, there is a built-in macro `offsetof`. The `std::array` container is your friend here, because its layout is identical to a sequence of identical variables.

---
A possible surprise is that using `sizeof` on a class can return a greater size then the end of the last member, because of the class' alignment.

---
### Multiple inheritance
In case of multiple inheritance, the parent classes are saved in the order they are listed, followed by the child's members.

```C++
struct T : std::string /*offset 0*/, std::vector<int> /*offset 32*/ {
	int a; // offset 56
}; // Don't do this! STL classes don't have virtual destructors!
```

---
### Polymorphism
A polymorphic class has a pointer to its vtable at offset 0, so the first element starts at offset 8 (on 64-bit architectures).

Virtual inheritance causes ancestor classes to be placed differently and accessed through vtable, making it far less predictable.

---
### Exercise
Create a structure that allows accessing several named `int` members through array index usage. Do it without implementing any methods or operators.
```C++
A a;
a.b = 15;
a.a[1] = 2;
// now, a.b is 2, not 15
```

---
## Union
Union is like struct, but it has two important differences:
* All offsets are 0
* It cannot inherit or be inherited from

The following can be, with proper methods, used as a representation of a short string that can be hashed without iteration:
```C++
union ShortString {
	std::array<char, 8> letters;
	uint64_t together;
	//...
```

Default copy constructors of classes that consist only of primitive types act like `memcpy`, so there's no need to optimise those using unions.

---
### Anonymous structs and unions
This feature is very useful, but not a part of standard. All major compilers support it, though. Structs, classes and unions can contain unnamed structs, classes and unions. Due to being nameless, they cannot be accessed, but their members are accessed as if they weren't nested.
```C++
union A {
	uint32_t a;
	struct {
		uint16_t b1;
		uint16_t b2;
	};
};
A a;
a.b2 = 3;
```

---
### Exercise
Create a struct or union that can be used to represent a binary message that contains:
* A 16 bit identifier
* 4 boolean values informing about status
* Three 8 bit values
* A 16 bit value
* A enum that has 14 possible values
* A 32 bit checksum

It must be as short as possible (12 bytes ideally) and its contents must be obtainable as an array of bytes without any conversions.

---
## Data on stack
A lot of variables on the stack may be optimised out, but it obeys some rules:
* On most architectures, a variable declared lower is on a lower address (inverted compared to structs)
* All variables are allocated when the block is entered
* The variables' values are set on the lines where they are initialised

This has rarely any effect, but it's not entirely inconsequential.

---
On most architectures, it is possible for an object to detect if it's a member of a specific object (and supposed to work with it) or not. A member is initialised after the outer class' base class and on a higher address. An unrelated stack variable is initialised either later and on an lower address or earlier on a higher address.
```C++
struct StatusMessage : Message {
	Register<int> value; // can check if it's not used outside and what is its parent
	Register<std::string> extras;
}
```
---
Allocating a huge object on stack will cause a stack overflow when the block is entered, making it unintuitive for debugging:
```C++
{
	std::string prefix = "av"; // Stack overflow here
	int initial = sqrt(value);
	std::array<std::array<std::string, 512>, 512> variations;
	// blablabla
```

---
## Custom smart pointers
Smart pointers are so useful that their use is almost imperative. However, the standard ones don't always suit our needs, for example in these cases:
* PIMPL
* Copy on write
* Thread safety
* Null pointer exception

---
A custom smart pointer acting like a unique pointer needs:
* A null pointer initialising default constructor
* A move constructor leaving null pointers behind
* A deleted copy constructor
* A move assignment operator leaving null pointer behind and conditionally deleting old contents
* A deleted copy assignment operator
* A destructor conditionally deleting contents
* An overloaded `bool` operator that returns if the value is not `nullptr`

---
A custom smart pointer acting like a shared pointer needs:
* A reference counter that starts at 1
* Overloads that guarantee that with each copy increments the reference counter
* Overloads that guarantee that each destruction or overwrite decrements the reference counter and deletes the object if it reaches zero

This is more complex than if it was acting like a unique pointer, but it usually allows greater convenience, as it allows the object to be moved around without regard for the ownership or deallocation.

An example of using a custom smart pointer for a copy-on-write string can be found [here](https://repl.it/repls/FoolhardyCluelessWaterfall).

If you need additional operators (more ways of dereferencing or something), you may overload the rarely used operator `->*` with an empty custom type.

---
### Exercise
Write your own implementation of a shared pointer, but with these key differences:
* The reference counter is allocated together with the object (does not support weak pointers as a consequence)
* Throws exceptions when dereferencing null pointers

---
## Endianness
```C++
union {
	std::array<char, 4> text;
	uint32_t numeric;
} converter;
converter.numeric = 0;
converter.text[0] = 65; // 'A'
```
Will `converter.numeric` be 0x65 or 0x65000000?

Most likely, it will be the 0x65, because little endian is used far more than big endian. But it shouldn't be taken for granted.

---
Endianness is usually very annoying to deal with. Without C++20, it cannot be detected at compile time.

It rarely matters, in most cases when an array is cast to a larger type, the larger type is only compared and cast back.

The problem unfolds when binary data are loaded to memory after being saved to disk in a cross-platform format or sent by another device.

---
## Copy elision
```C++
	std::string response = "Result is ";
	response += std::to_string(value);
	return response;
}
```
Since C++11, the variable `response` is not copied or moved when the function returns. It is already created at the return address. This isn't mandatory, but in practice, it always happens for anything that isn't a primitive type.

This means that returning large objects does not slow down the program.

This can be abused to allow a function to learn its return address.

---
Copy elision implies that move constructors or copy constructors may not be called when a value is returned. Since C++20, it is possible to return values that are neither movable nor copyable as long as the object is constructed on the return line:
```C++
return "Result is " + std::to_string(value);
```

---
C++17 adds structured bindings, making it possible to return more values, rendering output arguments absolutely useless.
```C++
std::tuple<std::string, std::string, std::string> parseLine(SourceFile);
//...
auto [id, name, description] = parseLine(source);
```

```C++
std::tuple<std::string, std::string, std::string> parseLine(SourceFile);
//...  Longer, harder to read, same speed
std::string id;
std::string name;
std::string description;
parseLine(source, id, name, description);
```

---
## Homework
Implement a class that facilitates parsing of binary messages. The messages contain bit flags, 1 byte unsigned integer values and 2 byte unsigned integer values. The class must be used this way:
```C++
static struct : Message {
	Flag online = 0;
	Flag busy = 1;
	Flag error = 2;
	Int1Byte chipTemperature = 8; // starts at bit 8
	Int2Byte presetVoltage = 16; // bits 16-32
	Int2Byte voltage = 32;
} status;
status.from(received);
if (status.online) //...
```
There are more ways to do this if you assume that the messages will have no more than 256 values and will not contain the byte sequence 0xFEDCBA9876543210 (this is an exercice, in a real application, this would be a security vulnerability).
