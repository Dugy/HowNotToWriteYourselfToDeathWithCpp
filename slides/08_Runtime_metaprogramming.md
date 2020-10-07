# 8. Runtime metaprogramming
Lambda functions are very very powerful.
* A lambda can capture other lambda functions and call them in sequence or conditionally
* The lambda functions captured by a lambda can be held in a `std::function`, allowing multiple lambda objects to be held in the same variable

This allows us to set the lambdas in a way that can emulate a Turing machine, making the lambdas Turing complete.

It shouldn't be used when performance is paramount, but it's still significantly faster than Python.

---
## Interpreter
If a configuration file is so complex that it needs to be used as an imperative programming language, it's either badly designed or a programming language should be used. Because no rule is inflexible, serious reasons to create an interpreter can happen.

---
It may happen that the program might need the user to supply formulae for some calculation but not anything more complex. The formulae take a certain group of input variables whose values are provided by the program and directly calculate the result in a single line.

`x * sin(angle + start_angle) + y`

Parsing the formulae is an algorithm that will not be delved into. If you were going to implement it, better don't simplify the task by requiring Polish notation. It is easier to parse, but atrocious to write.

To keep this example simple, each formula receives the variables as a an unordered map. Replacing the variable names with indexes when preparing and then calling it with a C-style array would make it run faster.
```C++
using Arguments = const std::unordered_map<std::string, double>&;
using Formula = std::function<double(Arguments)>;
```

---
```C++
Formula constant(double value) {
	return [value] (Arguments args) {
		return value;
	};
}

Formula variable(std::string name) {
	return [name] (Arguments args) {
		return args.at(name);
	};
}

Formula sine(Formula angle) {
	return [angle] (Arguments args) {
		return sin(angle(args));
	};
}

Formula addition(Formula first, Formula second) {
	return [first, second] (Arguments args) {
		return first(args) + second(args);
	};
}
```

---
### Exercise
Extend the example to have it support also context specific functions with any number of arguments.

```C++
using Arguments = const std::unordered_map<std::string, double>&;
using CustomFunctions = const std::unordered_map<std::string, std::function<std::vector<double>>>&;
using Formula = std::function<double(Arguments, CustomFunctions)>;
```

---
## What is a lambda internally
This lambda:
```C++
int added = 3;
auto add = [added] (int other) {
	return other + added;
};
```
Is equivalent to this:
```C++
int added = 3;
struct {
	int added;
	int operator() (int other) const {
		return added + other;
	}
} add = { 3 };
```
Except that the member `added` is inaccessible from anywhere else but the `opearator()` (but it's not really `private`).

---
### How `std::function` works
This is not a working or standard-compliant implementation, it just illustrates how it works:
```C++
template<typename Func>
struct function {
	using rettype = typename return_type<Func>::type; // Not implementing these here
	using arguments = typename argument_type<Func>::type...;

	struct Payload {
		virtual rettype call(arguments... args) = 0;
	}
	Payload* payload; // this is deepcopied in copy constructor and properly destroyed

	rettype operator() (arguments&&... args) {
		content.call(args...);
	}
	template <typename From>
	function(From from) {
		struct PayloadSpecific : Payload {
			From contents;
			rettype call(arguments... args) override {
				return From(args...)
			}
			PayloadSpecific(From&& from) : contents(from) {
			}
		};
		payload = new PayloadSpecific(from);
	}
//...
```

---
## Closure
A lambda's capture area determines which variables are held inside it and how:
* A copy - useful for callbacks
* A reference - useful when the lambda doesn't outlive its context

Examples of syntax:
* `[&]` captures everything mentioned inside by reference
* `[=]` captures everything mentioned inside as a copy
* `[this]` captures pointer to `this`, giving access to members
* `[*this]` captures a copy of `this`, can safely outlive `this` (C++17)
* `[&, i]` captures `i` as copy and everything else as reference
* `[=, &data]` captures everything as copy, but `data` as reference

---
It is possible to capture other lambdas. Under normal circumstances, a lambda cannot capture itself, because a class cannot contain itself.

It is however quite useful for GUI callbacks that reconstruct parts of the GUI and set themselves back as callbacks. This limitation can be worked around:
```C++
std::shared_ptr<std::unique_ptr<std::function<int()>>> capture
		= std::make_shared<std::unique_ptr<std::function<int()>>>();
*capture = std::make_unique<std::function<int()>>([capture, blabla] () {
	// some stuff
	(**capture)(); // Can call itself
	return stuff;
});
```

There is, however, a little problem with circular reference that requires explicitly resetting the `unique_ptr`.

---
### Exercise
Create a function that takes a lambda as argument and returns a shared pointer to an object that will call the lambda when destroyed.

---
### Generalised lambda capture
Since C++14, there is a shorter and more efficient way to insert member variables to lambdas:
```C++
auto getCounter() {
	return [count = std::make_shared<int>()] () {
		(*count)++;
		return *count;
	};
}
```

The part on the right can be any expression. The variable is created as if it was declared `auto`.

---
#### Exercise
Create a logger lambda that can be copied, but all copies share the same log file.

---
### Mutable lambda
A typical lambda's overload of `operator()` is a const method. This still allows editing variables captured through pointers and references, so in most cases, it's enough.

In order to turn a lambda into an object capable of changing its internal state, it must be declared `mutable`. This causes the overload of `operator()` to be nonconst.
```C++
auto getCounter() {
	return [count = 0] () mutable {
		count++;
		return count;
	};
}
```

`std::function` handles this fine, but when using template matching to get the argument types, it has to be accounted for. Of course, if a mutable lambda becomes const, it cannot be invoked.

---
### Lambda inheritance
Lambda is a class, so it can be inherited from. This can be useful to create a lambda-like object that has more overloads of `operator()`, resolved by different argument counts or types.
```C++
char ending = '\n';
auto printInt = [ending] (int num) {
	std::cout << "int:" << num << ending;
}; // its type is decltype(printInt)
auto printFloat = [ending] (float num) {
	std::cout << "float:" << num << ending;
};
struct Both : decltype(printInt), decltype(printFloat) { };
Both both{ printInt, printFloat };
both(3);
both(4.3f);
```

Both lambdas contain their copies of `ending`. The `Both` class can't be properly wrapped into `std::function`, so if this is to be used outside of the context where the lambdas are defined, it might need a custom reimplementation of `std::function`.

---
## Homework
Use a GUI framework (for example Qt) to create a function that takes a shared pointer to a data structure and returns a widget containing more widgets with callbacks that allow the user to modify the data structure.

You can choose what data structure it modifies, but it must be possible to add and remove elements at any location.
