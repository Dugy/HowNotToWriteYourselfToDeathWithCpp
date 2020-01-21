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
If a class serves to give some kind of managed access to another class, accessing the variable inside can be shortened to a mere `->` operator. The most known case of this are smart pointers.

```C++
if (plasmaMirror->temperature() > settings.maxTemperature()) {
```
The `plasmaMirror` variable's contents holds a class representing a device. When the `->` operator is called, the variable can be internally replaced with up-to-date values and then returned, so that nobody would forget calling some kind of update method (and not to write them everywhere).

---
It can be used to track updates to some value.
```C++
if (plasmaMirror->canMoveX())
    plasmaMirror.edit()->setOrientationX(plasmaMirror->orientationX() - change);
```
In this case, the `plasmaMirror` variable is some class wrapping a class, giving const access to it using the `->` operator and a non-const access using the `edit()` method that does some side-effect, for example marking some config file as dirty and in need of update when the program exits.

---
The dereference operator is a rather special one because it also eats up the dot operator on the class it outputs. Therefore, it's necessary for it to return a pointer to a class or something that has an overloaded dereference operator itself.
```C++
PlasmaMirror* operator->() {
    magnetohydrodynamicsSettings->dirty = true;
    return _plasmaMirror.get();
}
```
It's often useful to do this in a generic class rather than a class specific for every class.

---
An auxiliary object's destructor can be used to commit the changes after they are done.
```C++
struct Callback {
    std::function<Robot*()> access;
    std::function<void()> reaction;
    Robot* operator->() {
        return access();
    }
    ~Callback() {
        reaction();
    }
};
Callback operator->() {
    return Callback {
        [this] { return robot; },
        [this] { robot->uploadValues(); }
    };
}
```
```C++
robot->orientation[Orientation::X] = plan.now().x;
```

---
### Exercice:
Write a class that holds another class and counts the numner of times it is accessed.

```C++
properties->speed = 12;
properties->velocity = 12;
std::cout << properties.timesAccessed() << std::endl;
```

## Assignment operator
_TBA_