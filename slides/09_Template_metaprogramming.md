# 9. Template metaprogramming
C++ allows writing programs that decide what code will be generated. Unlike normal programs, they don't work with program input or user interaction, but they take template arguments and type names as input. All use cases are in the source code and errors almost never lead to compilable code.

This allows C++ programs to have the flexibility of dynamically typed languages, but without the performance costs or need for large numbers of tests.

---
## Compile time condition
There are multiple ways to implement a condition. Each of them may be suitable in different circumstances.

C++17 adds the usually most convenient way to do it:
```C++
template <typename Returned, typename Class, typename... Args>
void report(Class owner, Returned(Class::*func)(Args...)) {
	//...
	if constexpr(std::is_void_v<Returned>) {
		(owner->*func)(args...);
	} else {
		result.serialise((owner->*func)(args...));
	}
	//...
}
```

---
Function overloads can do the job as well:
```C++
template <typename Class, typename... Args>
void report(Class owner, void(Class::*func)(Args...)) {
	//...
	(owner->*func)(args...);
	//...
}
template <typename Returned1, typename Returned2, typename Class, typename... Args>
void report(Class owner, std::pair<Returned1, Returned2>(Class::*func)(Args...)) {
	//...
	auto returned = (owner->*func)(args...);
	result[0].serialise(returned.first);
	result[1].serialise(returned.second);
	//...
}
```

---
Template specialisation is the most powerful of them all, but also the least convenient.
```C++
template <typename Returned, typename Class, typename... Args>
struct Report {
	void report(Class owner, void(Class::*func)(Args...)) {
		//...
		result.serialise((owner->*func)(args...));
		//...
	}
};
template <typename Class, typename... Args>
struct Report<void, Class, Args...> {
	void report(Class owner, void(Class::*func)(Args...)) {
		//...
		(owner->*func)(args...);
		//...
	}
};
template <typename Returned1, typename Returned2, typename Class, typename... Args>
struct Report<std::pair<Returned1, Returned2>, Class, Args...> {
	void report(Class owner, std::pair<Returned1, Returned2>(Class::*func)(Args...)) {
		//...
		auto returned = (owner->*func)(args...);
		result[0].serialise(returned.first);
		result[1].serialise(returned.second);
		//...
	}
};

``` 

---
### Exercise
Write a class `is_addable` that checks if the type in the first argument can be added to the second argument.
```C++
bool canAddInt = is_addable<int, int>::value;
bool canAddIntToString = is_addable<std::string, int>::value;
```

---
## Static assert
Type errors can get through multiple template function calls, causing long backtraces even in case of petty errors. This can be prevented using `static_assert` which stops the compilation of a function or class and causes the compiler to write an error message with a backtrace up to the point where the `static_assert` failed.
```C++
template <typename T>
void addComponent(const T& added) {
	static_assert(std::is_same_v<decltype(added.id())>), std::string>,
		"Object must have an id() method that returns a std::string");
```

---
## Compile time loop
There is no compile time equivalent to `for` or `while`. There are two alternatives:
* recursion
* argument pack operations

Recursion is somewhat more useful, but each of them has use cases where the other one can't be used.

---
### Recursion
```C++
template <int num>
struct Factorial {
  static constexpr int factorial = Factorial<num - 1>::factorial * num;
};
template<>
struct Factorial<0> {
 	static constexpr int factorial = 1;
};
```

```C++
template <int power>
struct StaticPower {
	static double pow(double num) {
		return num * StaticPower<power - 1>::pow(num);
	}
};
template <>
struct StaticPower<0> {
	static double pow(double num) {
		return 1;
	}
};
```

---
### Argument pack operations
```C++
template <int... first>
struct Summer {
	template <int... second>
	static std::array<int, sizeof...(first)> sums() {
		return { (first + second)... };
	}
};
```

This kind of iteration transforms argument packs into argument packs, it cannot obtain a single result from multiple elements. C++17 adds folding expressions, which allows doing something like that. There is also a trick for turning single arguments into argument packs which is a part of the standard library.

---
#### Folding expressions
This part is C++17 only. The following example returns a sum of its template arguments.
```C++
template <int... first>
int sum() {
	return (... + first);
}
```

---
#### Creating argument packs
This usually needs to call an auxiliary function that gets the argument pack as an automatically deduced argument. It can also be a default argument. The important helpers are `std::make_index_sequence` that takes the size of the pack to be created and `std::index_sequence` that has the indexes as an argument pack which can be used to perform operations.
```C++
template <typename Made, typename... Args>
class LateMaker {
	std::tuple<Args...> pack;
	
	template<size_t... Indices>
	Made makeInternal(std::index_sequence<Indices...>) {
		return Made(std::get<Indices>(pack)...);
	}
public:
	LateMaker(const Args&... args) : pack(std::make_tuple(args...)) { }
	
	Made make() {
		return makeInternal(std::make_index_sequence<sizeof...(Args)>());
	}
};
```

---
### Exercise
Create a function that can be used to [calculate the greatest common divisor of two numbers](https://en.wikipedia.org/wiki/Euclidean_algorithm) at compile time.

---
### Exercise 2:
Use `if constexpr` and recursion to create a function that sorts its arguments (given by reference). Use bubblesort for simplicity.
```C++
// Variables lowest, lower, upper, highest are in undefined order
staticSort(lowest, lower, upper, highest);
// Now lowest < lower < upper < highest
```

---
## Side effects
So far, template metaprogramming seemed completely functional, where result depended only on input.

However, there is a way to get around it and cause compilation to have side effects. It's not a feature of the language, but rather an unexpected consequence of the standard. It can be used to access and assign global variables during the run of compilation from anywhere.

It's unlikely to be ever used by accident and multiple assignment is not possible, so it's quite safe to work with. The possibility is subject to defect report 2118, but no work is being done to change it as there've been no good suggestion what to do with it.

---
The trick is that the function can be forward declared and defined as a friend method of a template wherever it's instantiated.

```C++
int compiledA();

template <int num>
struct A {
  friend int compiledA() {
    return num;
  }
};

// this can be pretty much anywhere
A<11> a;

// ...
int done = compiledA(); // will get 11
```

---
In most cases, we'd like to use arrays. Forward declaring them is a bit tricky.
```C++
template <int N>
struct Index {
  friend constexpr int access(Index<N>);
};

template <int index, int num>
struct A {
  friend constexpr int access(Index<index>) {
    return num;
  }
};

// elsewhere
A<0, 12> a;

// ...
int atIndex0 = access(Index<0>()); // will get 12
```

---
## Turing completeness of compile-time metaprogramming
Because C++ templates can be used to implement both conditions and loops, they are Turing complete. This wasn't an intentional feature. It has several consequences:
* The syntactic correctness of a C++ program is an undecidable problem
* Any algorithm can be executed in compile time (a game of Tetris and a raytracer have been implemented already)
* Complexity is relevant also to compilation times

Its internal logic (but not syntax) is similar to Haskell. Because Haskell's syntax is better suited for this kind of programming, knowing how to write algorithms in Haskell is useful for C++ template metaprogramming.

---
## Limitations of template metaprogramming
Compile time operations cannot work with:
* floating point numbers - can be zero shifted or expressed as pairs of numerator or denominator
* strings - can be processed as packs of `char` variables, but there is no way to deal with a string in double quotes, so a last resort option is to write it as `toUpper<'h', 'e', 'l', 'l', 'o'>()`
* pointers - rarely needed, but if pointer arithmetic was available at compile time, it could be used to learn the system's endianness at compile time

Of course, it is possible to use compile time algorithms to generate code that uses strings, pointers and floating point numbers.

---
## Other tips
Templated usings and template globals can be used to shorten the usage of some template tools:
```C++
template<typename T>
using is_short_int = std::is_same<T, short int>;

template<typename T>
constexpr bool is_short_int_v = std::is_same<T, short int>::value;
```

---
Auxiliary class namespaces can be used to get multiple variadic parametres or even multiple argument packs.
```C++
template <typename... Args1>
struct FirstPack {
	template <typename... Args2>
	struct SecondPack {
		static auto multiplyPairs(Args1... args1, Args2... args2) {
			return (... + (args1 * args2));
		}
	};
};
```
Of course, this prevents deduction of parametres, so it is useful mostly in internal functions.

---
Auxiliary classes can be used to import additional template arguments into a function.
```C++
template <typename... Types>
struct TypeCarrier { };

template <typename... Types1, typename... Types2>
auto multiplyPairs(TypeCarrier<Types1...>, Types2... args) {
			return (... + (Types1(args)));
}
//...

return multiplyPairs(TypeCarrier<int, int>{}, width, length, height);
```

There is no way to explicitly write the function's template parametres in this case.

---
C++17 allows implicitly declaring class template parametres. It does not require any specific syntax, the template parametres are deduced so that a constructor could be called or the list of members matched. This helps avoid auxiliary functions to deduce template arguments when creating a class.
```C++
template <typename... Types1, typename... Types2>
auto multiplyPairs(std::tuple<Types1...> args1, Types2... args2) {
//...

multiplyPairs(std::tuple{1, 2, 3}, 1, 2, 3);
```

---
## Homework
Implement a class named `overloaded_function` that acts like `std::function`, but can be composed of any number of lambdas that act like overloads.
