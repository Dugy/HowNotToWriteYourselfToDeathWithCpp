# 6. Generic classes
The syntax for generic classes is similar to generic functions.
```C++
template <typename First, typename Second, typename Third>
struct Triplet {
	First first;
	Second second;
	Third third;
	std::tuple<First, Second, Third> tuple() const {
		return {first, second, third};
	}
};
```

Like generic functions, they must be entirely defined in headers. Auxiliary classes are needed to hide dependency-heavy code in source files.

---
Before C++17, class template arguments are not derived and have to be written explicitly. Functions can be used to derive them:
```C++
template <typename First, typename Second, typename Third>
Triplet<First, Second, Third> makeTriplet(First first,
		Second second, Third third) {
	return {first, second, third};		
}
//...
auto made = makeTriplet(engine, wheels, transmission);
```

---
## Generic classes and polymorphism
While virtual methods can't be generic, generic classes _can_ have virtual methods. These virtual methods of course can't be generic.

```C++
template <typename First, typename Second, typename Third>
struct Triplet {
	First first;
	Second second;
	Third third;
	virtual void activate() const = 0;
};
```

---
This can be very useful at avoiding unnecessary code. The main use case is to create a non-template parent with virtual methods and create a generic class that implements them. As a result, one doesn't have to implement the specific implementations.
```C++
struct Printable {
	virtual std::string print() const = 0;
};
template <typename T>
struct TypicalPrintable : Printable {
	T value;
	std:string print() const override {
		return std::to_string(value);
	}
};
```

---
In some more complex use cases, a generic class between the interface and the implementation can help avoid repetitive code.
* The interface class provides some basic functionality and type-agnostic interfaces
* The generic class inherits from the interface and implements shared type-dependent functionality
* The descendant class inherits from a specialisation of the generic class and implements specific functionality

```C++
struct RemoteInt : RemoteGeneric<int> {
	std::string describe() override {
		return _description + "(int)";
	}
}
```

---
### Exercice
Create a vector-like container that can store its contents on disk. It should have a type-independent interface for file access and type-dependent access to elements.

---
## CRTP
Curiously Recurring Template Pattern (CRTP) is an emergent feature arising from the lexing order that allows a class to inherit from a generic class instantiated to itself.
```C++
class FusionReactor : public ComplexDevice<FusionReactor> {
	//...
```

The generic parent can safely cast itself to the descendant and use anything within it, while the descendant can use anything from the parent using inheritance.

---
Because CRTP allows the parent to use pieces of its descendant's code, it is similar to some extent to virtual methods. There are some important differences, though.

Advantages of CRTP:
* Can check for existence and call non-virtual methods of the descendant
* Faster (can be inlined)
* Less code in the descendant (it can learn some additional information using template tricks)

Disadvantages of CRTP:
* Cannot be used as an interface (but it can use the template argument to implement an interface)
* More complicated to implement the parent
* Difficult to allow descendants of the CRTP inheriting class to use the CRTP

---
### Exercice
Create a class that allows its descendants to use CRTP get a method that returns their size.
```C++
struct VertexChunk : SizeAware<VertexChunk> {
//...
VertexChunk chunk;
int size = chunk.size(); // implemented in SizeAware
```

---
## Template specialisation
A generic class can have additional implementations designed to be more suited to specific types. Unlike functions, they don't have to be completely specialised.
```C++
template <typename First, typename Second, typename Third>
struct Triplet {
	First first;
	Second second;
	Third third;
};
template <typename Third>
struct Triplet<bool, bool, Third> {
	bool first : 1;
	bool second : 1;
	Third third;
};
```

---
It is of course possible to fully specialise the template.

```
template <>
struct Triplet<bool, bool, bool> {
	bool first : 1;
	bool second : 1;
	bool third : 1;
};
```
The more specialised version will take precedence over the less specialised one.

---
### Exercice
Create an imitation of `std::optional` that is specialised to take only 1 byte if it contains a `bool`. 

---
## SFINAE with template class specialisation
Template class specialisation is in a way similar to function overloads. It's similar enough that it's also possible to use SFINAE to cause a particular specialisation to fail if some complex conditions aren't met.
```C++
template <typename Compressor, typename SFINAE>
struct Compress {
	constexpr static bool valid = false;
};
template <typename Compressor>
struct Compress<Compressor, std::enable_if_t<std::is_base_of<Zip, Compressor>::value>> {
	constexpr static bool valid = true;
	
	static Data compress(Data value) {
		return Compressor::compress(value);
	}
};
// ...
static_assert(Compress<Tar, void>::valid, "Tar archiver not included!");
```

---
Template specialisation SFINAE has some advantages over function overloading SFINAE:
* Implementing a base case that applies if all others fail is always possible
* New specialisations to new types can be implemented elsewhere, making it possible to design libraries to be specialisable to support custom types
* Failures to match can be reported with clear error messages using `static_assert`

However, it is less convenient to use.

---
### Exercice:
Create a `vectorify()` function that converts containers `std::unordered_map` and `std::list` into `std::vector`. In case of containers that contain pairs of indexes and elements, it should create vectors of pairs. Make it extendable with other types. It must return a custom error message if used on an unsupported type.

---
## Recursion
Variadic templates work with class templates as well.
```C++
template <typename... Ts>
class Heterogenous {
    std::tuple<Ts...> _contents;
public:
    template <int index>
    auto& at() {
        return std::get<index>(_contents);    
    }
};
```

---
And it can be used to implement recursion:

```C++
template <typename T, typename... Ts>
class Heterogenous {
    T _contained;
    Heterogenous<Ts...> _parent;
public:
    template <int index>
    auto& at() {
        if constexpr(index == 0)
            return _contained;
        else
            return _parent.template at<index - 1>();
    }
};

template <typename T>
class Heterogenous<T> {
    T _contained;
public:
    template <int index>
    auto& at() {
        static_assert(index == 0, "Index at Heterogenous is too high");
        return _contained;
    }
};
```

---
Remarks:
* If a template method calls another template method, its type might not be known the time and the `<` template braces `>` will be mistaken for comparison operators, resulting in a syntax error; in that case, it's necessary to write the keyword `template` before the method name
* The template class needs to be partially specialised for the base case (to stop recursion before an error)
* Template argument iteration is possible in many cases, but the recursive approach is usually easier

---
# Homework:
Create a class that holds more values of different types. Converting it to one of those types yields the value of that type.
```C++
auto path = graph.findPath(start, end);
if (path) {
	int length = path;
	std::vector<Node> steps = path;
//...
```
