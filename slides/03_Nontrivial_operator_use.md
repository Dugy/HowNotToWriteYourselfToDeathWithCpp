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

Here is an even worse case:
```C++
device.setVoltage(newValue); // boring
device,setVoltage,newValue; // everyone likes puzzles
// Operator comma with the custom global variable type returns a class with overloaded comma operator
```

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

Actual code:
```C++
struct {
    Robot* parent;
    void operator() () {
        std::lock_guard lock(parent->mutex);
        parent->stopAttackingHumans();
    }
} response;
response.parent = this;
safetyButton.setCallback(response);
```

Shortened by lambda:
```C++
safetyButton.setCallback([this] {
    std::lock_guard lock(mutex);
    stopAttackingHumans();
});
```

---
Functors are often expected as function arguments to functions that expect something to be called from their bodies. Here, suitable lambdas are faster to write and faster to execute than classes implementing an interface that contains the function.
```C++
data.erase(std::remove_if(data.begin(), data.end(), [] (Value* val) {
    return val->errors() != std::string::npos;
}));
```

---
Like all functions, overloaded operators can have overloads for different argument types. Unlike any other operator, the function call operator can have any number of arguments.
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
### Exercise:
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

If you need a dereference-like operation that takes an argument, you can overload the `->*` operator (its precedence is rather low, may need extra parentheses if something is called on the result).

---
The dereference operator is a rather special one because it also recursively dereferences its outputs until it finds something the `->` operator can't be applied to. Therefore, it's necessary for it to return a pointer to a class or something that has an overloaded dereference operator itself.
```C++
PlasmaMirror* operator->() {
    _settings->dirty = true;
    return _plasmaMirror.get();
}
```
It's often useful to do this in a generic class rather than a class specific for every class.

---
An auxiliary object's destructor can be used to commit the changes after they are done.
```C++
struct RobotController {
    struct Robot {
        //...
            void uploadValues() {
        //...
        }
    };
    std::shared_ptr<Robot> _robot;
    //...
    struct Callback {
        std::function<std::shared_ptr<Robot>()> access;
        std::function<void()> reaction;
        std::shared_ptr<Robot> operator->() {
            return access();
        }
        ~Callback() {
            reaction();
        }
    };
    Callback operator->() {
        return Callback {
            [this] { return _robot; },
            [this] { _robot->uploadValues(); }
        };
    }
};
```
```C++
robotController->orientation[Orientation::X] = plan.now().x;
```

---
### Exercise:
Write a class that holds another class and counts the number of times it is accessed.
```C++
properties.resetTimesAccessed();
properties->speed = 12;
properties->velocity = 12;
std::cout << properties.timesAccessed() << std::endl; // Should print 2
```

---
## Assignment operator
As you all probably know, the assignment operator is commonly overloaded for custom copying or moving objects into existing ones.
```C++
void operator=(const std::string& newName) {
    _name = newName;
}
```
However, it has plenty of other uses.

---
Suppose that structures like this are used extensively in the code:
```C++
settings.location.coordinates = coordinates;
settings.userLocation.city = city;
settings.time.zone = timeZone(city);
```
However, the paths to nested objects are annoyingly long, so it would be more convenient to write only:
```C++
settings = coordinates;
settings = city;
settings = timeZone(city); // Could also be derived after the previous assignment
```
But there can't be a huge class containing all those settings because of the difficulties caused by violations of the Single Responsibility Principle.

---
A possible way to do it is to make the class inherit from `Location` and other classes that can be used for settings and inherit their assignment operators.
```C++
    using Location::operator=;
```

If that's not feasible (for example if the class is used multiple times), the assignment operator can have a custom overload:
```C++
Settings& operator=(const Location& location) {
    coordinates = location.coordinates;
    navigationDatabaseEntry = location.navigationdatabaseEntry;
    altitude = location.altitude;
    return *this;
}
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

Note #2: User-defined conversion operators do not apply transitively, but conversions between integer and floating point types can apply afterwards.

---
It can be used to create an imitation of another type that silently performs additional operations when used:
```C++
class FakeInt {
    int value;
    int min = 0;
public:
    operator int() {
        if (value < min)
            throw std::runtime_error("Value too low");
        return value;
    }
    //...
};
```
This will be implicitly converted to `int` when arithmetic operations are performed.

Be wary not to copy this into an `auto` variable, because you will end up with a copy of the object that will still act as `int`, possibly causing unwanted behaviour. You can make this class uncopiable.

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
    //...
};
```
This doesn't enable `operator +=`, `operator -=` and similar, but that isn't what you want to do, as every assignment has an invisible overhead.

---
### Exercise:
Create a class that can be used as a float, but yields a different random value in the range betwen 0 and 1 whenever used in arithmetic.
```C++
float upToTwo = unpredictable * 2;
float upToThree = unpredictable * 3;
```

---
## Operators new and delete
If you want some classes to be allocated differently than the standard allocator does, you can overload the `new` and `delete` operators. This may be needed for a variety of _niche_ reasons:
* Allowing a parent class to learn the size of the derived class that is created
* Indexing the objects
* Zeroing memory after delete, or filling it with invalid data
* Debugging some tough problems
* Allocating some objects in shared memory
* A garbage collector of some sort
* Some advanced optimisations

---
Any object of the desired type or even all objects will be allocated and deallocated differently than by simply calling `malloc()`. It also applies to objects allocated using `make_unique` or `make_shared`, because they internally use `new`. Containers like `std::vector` allocate blocks of memory and use placement constructors inside, so overloading the `new` operator for the class will _not_ change their allocation.

Also, it is not called any time the class is created on stack.

---
The overload is written as follows:
```C++
void* operator new(size_t size) // implicitly static
{
	void* address = malloc(size); // this uses the normal new  
	std::cout << "Allocating a Leafblower at: " << size << std::endl;
	return address;
}
void operator delete(void* pointer) // implicitly static
{
	std::cout << "Deallocating a Leafblower at: " << size << std::endl;
	free(pointer);
}
```

If it's not within a class, it applies globally, on the linker level.

---
## Homework
Create three classes that behave like integer, float and string. They all inherit from a common ancestor and override its methods for serialisation and deserialisation (JSON, XML, whatever you like). Create a class that can be used to initialise them all. They will use the same object to save their values when they're destroyed.

```C++
Electrocard::Electrocard(nlohmann::json& configuration) :
    configurator(this, configuration, "electrocard"),
    frequency(configurator, "freq"),
    maxVoltage(configurator, "voltage"),
    waveform(configurator, "wave"),
    runMode(configurator, "mode"),
```
