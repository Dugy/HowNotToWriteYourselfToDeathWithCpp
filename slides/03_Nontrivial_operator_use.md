# 3. Nontrivial operator use
Operator overloading is fairly straightforward and well known:
```C++
std::string operator*(const std::string base, int multiplier) {
    std::string made = base;
    for (int i = 1; i < multiplier; i++)
        made += base;
    return made;
}
```
They also shouldn't be used much because they can make the code difficult to read. Arithmetic operators should be overloaded only for classes that represent mathematical entities with such operations defined, unlike in the example above.

---
## Function call operator

If all of a class' usage revolves around a single method, making that method more important than the class itself, it can make sense to use the method as if the class itself was a function. Such a class is called a _functor_.
```C++
Serialiser serialise;
serialise.setOutput(Serialiser::JSON);
serialise.setIndentation('\t', 1);
serialise(doughnuts, "doughnuts");
serialise(berliners, "berliners");
serialise(applePies, "applePies");
serialise(icecreams, "icecreams");
```

---
A lambda function is actually a functor, typically with a `const` function call operator. Because of their shorter syntax, they are often more convenient than explicit functors.
```C++
struct {
    Robot* parent;
    void operator() () {
        std::lock_guard(parent->mutex);
        parent->stopAttackingHumans();
    }
} response;
response.parent = this;
safetyButton.setCallback(response);
```
```C++
safetyButton.setCallback([this] {
    std::lock_guard(mutex);
    stopAttackingHumans();
});
```

---
Functors are often expected as function arguments to functions that expect something to be called from their bodies. Assuming they would be given classes with methods with some names would make them counter-intuitive.
```C++
data.erase(std::remove_if(data.begin(), data.end(), [] (Value* val) {
    return val->errors() != std::string::npos;
}));
```

---
Like all functions, operator overloads can be overloaded for different argument types. Unlike any other operator, the function call operator can have any number of arguments.
```C++
// Some class, named Serialiser or something
void operator() (std::string& value, const std::string& name) {
    serialiserChosen().setOptions(options);
    serialiserChosen().serialiseString(value, name);
}
void operator() (float& value, const std::string& name) {
    serialiserChosen().setOptions(options);
    serialiserChosen().serialiseFloat(value, name);
}
```

---
### Exercice:
Write a function that calls a functor when done.
```C++
fitAmbihelicalHexnuts(hexnuts, hints, [] (int seconds) {
    std::cout << "Ambihelical hexnuts fitted in " << seconds << " seconds" << std::endl;
});
```

---
## Dereference operator
_TBA_