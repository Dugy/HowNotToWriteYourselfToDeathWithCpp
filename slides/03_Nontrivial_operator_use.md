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
The `plasmaMirror` variable's contents may hold a class that represents a device. When the `->` operator is called, the variable can be internally replaced with up-to-date values and then returned, so that nobody would forget calling some kind of update method (and not to write them everywhere).

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

---
## Assignment operator
As you all probably know, the assignment operator is commonly overloaded for custom copying or moving objects into existing ones.
```C++
void rename(const std::string& newName) {
    _name = newName;
}
```
However, it has plenty of other uses.

---
Suppose that structures like this are used extensively in the code:
```C++
settings.location.coordinates = coordinates;
settings.user_location.city = city;
settings.time.zone = timeZone(city);
```
The circumstances are that it would be far more convenient to use this instead:
```C++
settings.coordinates = coordinates;
settings.city = city;
settings.zone = timeZone(city);
```
But you need to assign some kind of `Location` object into `settings.location` as well. Same for other subsets of settings.

---
A possible way to do it is not to make the class a composition and allow assigning those `Location` objects into the class itself:
```C++
Settings& operator=(const Location& location) {
    coordinates = location.coordinates;
    navigationDatabaseEntry = location.navigationdatabaseEntry;
    altitude = location.altitude;
    return *this;
}
```
This doesn't even have to be written explicitly if the `Settings` class inherits from `Location` and other classes that represent subsets of the settings:
```C++
struct Settings : Location, UserLocation, Time, HotDogBrand {
```

---
The assignment operator really shines when used in combination with the conversion operator.

---
## Conversion operator
The most common use of the conversion operator to allow conveniently checking if some state is valid:
```C++
struct Laser {
    //...
    operator bool() {
        return isInitialised() && powerGood() && authorised();
    }
};
```
```C++
if (laser) {
    laser.minChargup(10);
    laser.fire();
}
```
Note: The syntax is somewhat unusual, because the return value is specified after the `operator` keyword rather than before.

---
It can be used to create an imitation of another type that silently performs additional operations used:
```C++
class FakeInt {
    int main;
    int uses = 0;
public:
    operator int() {
        uses++;
        return main;
    }
};
```
This will be implicitly converted to `int` when arithmetic operations are performed.

Be wary not to copy this into an `auto` variable, because you will end up with two trackers that will both act as `int`. You can make this class uncopiable.

---
Together with the assignment operator, it can be used as a proxy to a variable that may need to be accessed in a special way (different format, mutex, different computer):
```C++
class FakeDouble {
    std::shared_ptr<int64_t> _proxy;
    double _scale;
public:
    operator double() {
        return *_proxy * _scale;
    }
    double operator=(double assigned) {
        *_proxy = assigned / _scale;
        return assigned;
    }
};
```
This doesn't enable `operator +=`, `operator -=` and similar, but that isn't what you want to do, as every assignment has an invisible overhead.

---
### Exercice:
Create a class that can be used as a float, but yields a different random value in the range betwen 0 and 1 whenever used in arithmetic.
```C++
float upToTwo = unpredictable * 2;
float upToThree = unpredictable * 3;
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
## Conversion constructor
While conversion operator converts a given class to various, often impossible to extend classes, the conversion constructor converts any classes to a given class.

The syntax is relatively common:
```C++
SuperPointer(const std::shared_ptr<T>& set) {
```
This one can be used to convert shared pointers implicitly to the `SuperPointer` class:
```C++
void processString(std::shared_ptr<std::string> stringValue) {
    SuperPointer<std::string> string = stringValue;
```
This will convert shared pointer to `SuperPointer`, so that the assignment operator would work on it.

---
Implicit conversions may apply recursively, which can lead to unwanted conversions to completely different classes. Use the `explicit` keyword with the constructor to prevent it.

To allow wide ranges of conversions, a templated version of conversion operator can be used. In that case, SFINAE can be used to prevent too many possible conversions, possibly leading to ambiguities.

---
## Homework
Create three classes that behave like integer, float and string. They all inherit from a common ancestor and override its methods for serialisation and deserialisation (JSON, XML, whatever you like). Create a class that can be used to initialise them all, setting an object where their properties should be saved when a class that contains them is serialised.

```C++
Electrocard::Electrocard(const nlohmann::json& configuration) :
    configurator(this, configuration, "electrocard"),
    frequency(configurator),
    maxVoltage(configurator),
    waveform(configurator),
    runMode(configurator),
```