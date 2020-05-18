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

Parsing the formulae is an algorithm that will not be delved into. If you were going to implement it, better don't simplify the task with Polish notation. It is easier to parse, but atrocious to write.

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
Except that the member `added` is inaccessible from anywhere else but the `opearator()`.

---
### How `std::function` works
This is not a working implementation, it just illustrates how it works:
```C++
template<typename Func>
struct function {
	using rettype = typename return_type<Func>::type; // Not implementing these here
	using arguments = typename return_type<Func>::type...;

	struct Payload {
		virtual rettype call(arguments... args) = 0;
	}
	Payload* payload; // this is copied in copy constructor and properly destroyed

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
