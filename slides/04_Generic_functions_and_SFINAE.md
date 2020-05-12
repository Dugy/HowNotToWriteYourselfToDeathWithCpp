# 4. Generic Functions and SFINAE
These allow you to write code once and apply it to all types.
```C++
auto secondPower(auto number) {
    return number * number;
}
```
This simple case replaces a macro that would, in case of an error, never show a proper line where the issue happened.

Note: `pow(number, 2)` may execute a universal algorithm for computing power, which is slower than multiplication (premature pessimisation).

---
To add more constraints to the types, templated types can be explicitly stated:
```C++
template <typemame T>
T secondPower(const T& number) {
    return number * number;
}
```

---
```C++
template <typename T>
bool isInVector(const std::vector<T>& vec, const T& element) {
    for (auto& it : vec)
        if (it == element)
            return true;
    return false;
}
```
The template arguments can be deduced by the compiler, so there is no need to specify what is `T`:
```C++
if (isInVector(needed, it)) {
```

---
There might be a need to specify the template arguments in the case for debugging purposes (the arguments' types are complex and it's hard to tell why they don't match):
```C++
if (isInVector<std::optional<CachedValue>>(cache, value)) {
```

Or if it's not a part of the argument list:
```C++
auto button = std::make_unique<QPushButton>();
```

---
Internally:
* It's like dynamic typing, but it detects errors at compile time and has no performance cost
* A generic function is a function template, not a real function
* It's not possible to obtain a pointer to a function template nor to put it into a separate compilation unit (which is the reason why C++ compiles so much slower than C)
* The function template is _instantiated_ when it's used with template parameters or arguments allowing to deduce them
* An instantiated function template is a regular function
* An instantiated template is very likely to be inlined (avoiding the cost of a function call and allowing better optimisation) because it's always available to the compilation unit (like other functions defined in headers)

---
**Exercice:** Write a generic function that removes an element at given index from a vector by moving the last element into its place and removing the last element (fast removal but without keeping the order).

```C++
fastRemove(actions, i);
```

---
## Parameter type deduction

In most cases, templating the whole type is enough:
```C++
template <typename T>
void toJson(const T& map) {
    const nlohmann::json& made;
    for (auto& it : map)
        made[it.first] = toJson(it.second);
    return made;
}
```
However, think of the error message if someone tries to call it on a vector...

---
But the template matching is quite smart and can do much better things:

```C++
template <typename T>
void toJson(const std::unordered_map<std::string, T>& map) {
    const nlohmann::json& made;
    for (auto& it : map)
        made[it.first] = toJson(it.second);
    return made;
}
```
And it doesn't only help finding errors:
```C++
template <typename T>
T midElement(const std::vector<T>& source) {
    return source[source.size() / 2];
}
```

---
Since C++14, it also can derive more types from a single argument:
```C++
template <typename IndexType, typename ValueType>
auto getNthElement(const std::map<IndexType, ValueType>& source, size_t order) {
    int at = 0;
    for (auto& it : source) {
        if (at < order)
            at++;
        else
            return it;
    }
    throw(std::out_of_range());
}
```
Note: if the function's content is known in the header, its return value can be deduced (needs the `auto` keyword). In some complex cases, almost the entire function code is needed to derive the type, making some generic functions' return types unreadable before C++14.

---
Functions and methods's arguments can be matched too:
```C++
template <typename ParentType, typename ReturnType>
std::function<ReturnType()> standalone(ReturnedType(ParentType::*method)(), ParentType* parent) {
    return [parent, method] () {
        return (parent->*method)();
    };
}
```
You can see the whole usage and try it for yourself [online](https://repl.it/repls/RoundedSlightAnalysts).

---
It can also match a template:
```C++
template <
    template<typename...> class Container,
    typename Content
>
std::pair<Content, Content> edges(const Container<Content>& data) {
  return std::make_pair(*data.begin(), *std::prev(data.end()));
}
```
```C++
std::pair<int, int> extremities(std::vector<int>& values) {
    std::sort(values.begin(), values.end());
    // edges(3); // Will not match any function rather than fail inside edges()
    return edges(values);
}
```
This can be used on multiple container types.

Note: `template<typename> class` would be enough for a single template argument class, but the standard containers take several other template arguments that are almost always left default-valued.

---
**Exercice:** Create a class that allows its descendants to bind its setters with a function call with a single parameter.
```C++
class Drill : public Binder {
    void setDepth(float depth);
    std::function<void(float)> depthSetter() {
        return bindSetter(&Drill::setDepth);
    }
};
```

---
## Variadic template
The `typename...` argument can stand for a comma-separated list of any number of arguments (including 0). It always has to be expanded with `...` when used:
```C++
class LaserWeldingRobot : public Robot {
    LaserType _laser;
public:
    template <typename... ParentArgs>
    LaserWeldingRobot(LaserType laser, ParentArgs... forParentConstruction)
        : Robot(forParentConstruction...),
        _laser(laser) {
    }
};
```
This can be useful if class `Robot` has multiple constructors, but we don't want to define a constructor for each in each derived class. You can see the code in context [here](https://repl.it/repls/RobustCuteSignature).

---
The number of arguments in an argument pack can be inferred using the `sizeof...` operator.
```C++
template <typename Arg, typename... Args>
auto makeArray(const Arg& arg, const Args&... args) {
    return std::array<Arg, sizeof...(Args) + 1>{arg, args...};
}
```
```C++
  auto active = makeArray(3, 5, 7, 9, 12, 17);
```
Note: in this case, the first argument is used to determine the type of the array and it will not work right if some argument types are not compatible. This function can be useful to avoid having to count the listed elements or to update the count when a new element is added. However, having to write both the size and all elements is a double-check against human errors.

You can try it for yourself [here](https://repl.it/repls/ShyLovelyScreencast).

---
While argument packs are convenient for passing groups of arguments to overloaded functions, they can be far more powerful when used recursively:
```C++
template <typename T>
T sum(T arg) { // Base case
    return arg;
}

template <typename T1, typename T2, typename... Others>
auto sum(T1 first, T2 second, Others... rest) { // Recursive step
    return first + sum(second, rest...);
}
```
```C++
    int total = sum(robot1.arms, robot2.arms, robot3.arms);
```
Note: like in any recursion, a base case is needed for the recursion to work. The functional overload providing the recursive step must take an additional argument from the pack in order to be distinguished from the base case.

---
**Exercice:** Implement your own `std::make_unique` function, which can come very handy if you have access to C++11, but not to C++14.
```C++
std::unique_ptr<std::string> str = std::make_unique<std::string>("Default content");
```
Notice that even if several first arguments are explicitly stated, the missing ones are left to be derived from arguments.

---
## if constexpr
C++17 has added a nifty new alternative to overloading of template functions. It acts similarly to a regular `if`, but is handled at compile time and decides which part of the code compiles and which doesn't (and can thus contain semantic errors).
```C++
template <typename T, typename... Others>
auto sum(T arg, Others... rest) {
    if constexpr (sizeof...(Others) >= 1)
        return arg + sum(rest...);  // Recursive step
    else
        return arg; // Base case
}
```
Notice: `sum()` cannot be called with no arguments, so the first branch cannot compile when `sum` is called with a single argument. However, that is not a problem, because `if constepxr` tries to compile that branch only if there is at least one argument.

---
It is possible to infer some properties of the types of the variables used:
```C++
template <typename T, typename... Others>
auto sum(T arg, Others... rest) {
    if constexpr (sizeof...(Others) >= 1) { // Recursive step
      if constexpr (std::is_arithmetic<T>::value)
        return arg + sum(rest...); 
      else {
        auto partial = sum(rest...);
        for (auto it : partial)
          arg.push_back(it);
        return arg;
      }
    } else // Base case
        return arg;
}
```
This will add numbers or join vectors, depending on the type.

---
There are other checks available
* `std::is_pointer<T>::value`
* `std::is_reference<T>::value`
* `std::is_const<T>::value`
* `std::is_class<T>::value`
* `std::is_void<T>::value`
* `std::is_same<T1, T2>::value`
* `std::is_base_of<Base, Derived>::value`
* ... (see [CppReference](https://en.cppreference.com/w/cpp/types) for more)
Many of these can be shortened by appending `_v` at the end of the check's name and skipping the `::value` part, like `std::is_base_of_v<Base, Derived>`, but not all.

---
Types can be transformed if necessary:
* `std::remove_const<T>::type` - outputs the same type as `T`, but not const
* `std::remove_pointer<T>::type`
* `std::decay<T>::type` - modifies the type of `T` to be passed by value
They can be shortened like `std::decay<T>::type` to `std::decay_t<T>`.

---
To obtain the result of an expression, use the `decltype` keyword:
```C++
if constexpr (std::is_base_of_v<Robot, decltype(*data.begin())>) {
```
It may be useful also elsewhere:
```C++
decltype(data.begin()) position = seeker.getIteratorTo(data, element);
```
If the expression in `decltype` is too long, you can store it:
```
using DataType = decltype(*data.begin());
```

---
**Exercice:** Write a variant of `std::max` that can take any number of arguments and returns the greatest one of them:
```C++
int max = maximum(3, 8, 19, value);
```

---
## SFINAE
_Substitution Failure Is Not An Error._ Ouch, that's so cryptic that maybe even Robert Langdon doesn't know what that is supposed to mean.

A function template may be conditionally prevented from being used by making one or more of its arguments semantically incorrect. The program itself remains correct.

---
This allows writing overloaded functions like these:
```C++
template <typename T>
void removeValue(T& container, decltype(std::declval<const T>()[0]) value) {
    for (unsigned int i = 0; i < container.size(); i++) 
        if (container[i] == value) {
            container.erase(container.begin() + i);
            i--;
        }
}
template <typename T>
void removeValue(T& container, decltype(std::declval<T>().begin()->second) value) {
    for (auto it = container.begin(); it != container.end(); )
        if (it->second == value) {
            it = container.erase(it);
        } else ++it;
}
```
If `T` is a vector, the second argument of the second overload is invalid and the first overload will be selected. If `T` is a map or unordered map, the second argument of the first overload will be invalid and the second overload will be selected.

---
`std::enable_if_t` is a valid type (the exact type is its optional second argument) if the condition in its first argument is true.
```C++
template <typename T>
T add(T first, std::enable_if_t<std::is_arithmetic<T>::value, T> second) {
    return first + second;
}

template <typename T>
T add(T first, std::enable_if_t<!std::is_arithmetic<T>::value, T> second) {
    for (auto it : second)
        first.push_back(it);
    return first;
}
```
This is an old compiler friendly version of `if constexpr`.

---
The type of the second argument is very difficult to read. It's better to use the SFINAE argument as an extra template parameter:
```C++
template <
    typename T,
    std::enable_if_t<std::is_arithmetic<T>::value>* = nullptr
>
T add(T first, T second) {
    return first + second;
}
```

---
Of course, the conditionally incorrect code can be used as the SFINAE argument itself:
```C++
template <typename T, decltype(std::declval<const T>()[0])* = nullptr>
```

---
**Exercice:** Use SFINAE and a JSON or a XML library or just a string to string unordered map to create a serialising function that serialises numbers and strings normally and smart pointers and `std::optional` to these types as optional keys.
```C++
made.serialise("id", id); // std::string
made.serialise("version", version) // int
made.serialise("author", author); // std::shared_ptr<std::string>
made.serialise("rating", rating); // float
made.serialise("comment", comment); // std::unique_ptr<std::string>
```

---
## Generic conversion operator
This is where it's possible to deduce the return value of a function.
```C++
struct MakeSmart {
    template<typename T>
    operator std::shared_ptr<T>() {
        return std::make_shared<T>();
    }

    template<typename T>
    operator std::unique_ptr<T>() {
        return std::make_unique<T>();
    }
};
```
```C++
std::unique_ptr<QPushButton> button = MakeSmart();
std::shared_ptr<std::string> edited = MakeSmart();
```
You can try it [online](https://repl.it/repls/IllCorruptCharacters).

---
The generic conversion operator allows using a single short symbol without template arguments to initialise a lot of different types, possibly supplying them with various arguments the class holds as members.

It is very prone to ambiguous function call issues, especially with MSVC that generally lacks in compliance to standards. You will need SFINAE to prevent the unwanted conversions from matching.

---
## Matching functions
The most common case will be dealing with instances of `std::function`:
```C++
template <typename Returned, typename... Args>
void grabFunction(std::function<Returned(Args...)> func) {
```

---
Free functions can be captured through the strange syntax of function pointers:
```C++
template <typename Returned, typename... Args>
void grabFunction(Returned(*func)(Args...)) {
```

Member functions can be captured similarly:
```C++
template <typename Object, typename Returned, typename... Args>
void grabFunction(Returned(Object::*func)(Args...)) {
```

---
Example of usage:
```C++
template <typename Object, typename Returned, typename... Args>
std::function<Returned(Args...)> exportMethod(Object* instance, Returned(Object::*func)(Args...)) {
	return [instance, func] (Args... args) {
		return (instance->*func)(args...);
	};
}
//...
std::string a = "Blablabla";
auto clearMyString = exportMethod(&a, &std::string::clear);
```

---
Lambdas are not functions, they are classes. They however always have an overloaded `operator()`, which is a method
```C++
template <typename T>
void grabFunction(T func) {
	return grabFunction(&func, &T::operator());
}
```

---
## Homework
Use `if constexpr` to create a function that sorts its arguments (given by reference). Use bubblesort for simplicity.
```C++
// Variables lowest, lower, upper, highest are in undefined order
staticSort(lowest, lower, upper, highest);
// Now lowest < lower < upper < highest
```
