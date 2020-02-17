# 7. Memory manipulation
These are little details that are usually several levels below the usual level of abstraction, but they can be handy from time to time. Here are a few use cases:
* Writing or reading binary data
* Some part is performance critical (and optimisation can be easier than parallelism)
* Reinventing the wheel (if the usual one doesn't fit your needs)

---
## Data in struct or class

---
## Custom smart pointers
Smart pointers are so useful that their use is almost imperative. However, they don't always suit our needs, for example in these cases:
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
* A deleted move assignment operator
* A destructor conditionally deleting contents

---
A custom smart poiter acting like a shared pointer needs:
* A reference counter that starts at 1
* Overloads that guarantee that with each copy increments the reference counter
* Overloads that guarantee that each destruction or overwrite decrements the reference counter and deletes the object if it reaches zero

This is more complex than if it was acting like a unique pointer, but it usually allows greater convenience, as it allows the object to be moved around without regard for the ownership or deallocation.

An example of using a custom smart pointer for a copy-on-write string can be found [here](https://repl.it/repls/FoolhardyCluelessWaterfall).