# 10. Putting it all together
Now, you possess a vast arsenal of tricks that can be employed to shorten repeating parts of code to necessary minimum. Even if you can't recall them all, you probably know where to look.

This chapter showcases some examples what can be achieved and how.

---
## Serialisation
```C++
struct LaserConfig : Serialisable {
	std::string address = key("address") = "192.168.4.14";
	int port = key("port") = 94;
	float wavelength_nm = key("wavelength_nm") = 330;
	float pulsePower_MW = key("pulsePower_MW") = 3.4;
};
```
The `key()` method stores some information in the `Serialisable` parent class that also provides methods for the serialisation/deserialisation itself. The code could be further shortened by macros (that can take an argument as both a variable name and a string literal), at the cost of even worse obfuscation.

---
There are several hurdles with the implementation:
* The initialising value is assigned into the output of the `key()` method
* The `key()` function seems to need multiple return types
* The `key()` function must somehow deduce the address of the object the value is assigned to and the parent address

They can be dealt with, though.

---
First, let's analyse what this line actually does in a class declaration:
```C++
std::string address = key("address") = "192.168.4.14";
```

If there was a constructor, one of the lines it its initialisation part would look like this:
```C++
address(key("address") = "192.168.4.14"),
```

Thus, the order is:
1. The `key()` function returns a value
2. The assignment operator on the value returned by `key()` is called
3. The member variable `address` is initialised with the result of the assignment

---
The `key()` function can be a member function of `Serialisable`. All parent classes are fully initialised when anything inside the child class is initialised. Therefore, it's ready for saving information about the members when it's needed.

---
Most of the magic lies in the return value of `key()`. It has to be an auxiliary class with some special functionality:
* It can be assigned anything and returns a similar type that also holds the assigned value
* It has an overloaded generic conversion operator that does several things
	* It sets the result either to a default initialised value or to the assigned value
	* It uses a `static_assert` to check if the result type can be initialised from the assigned value (or default initialised)
	* It checks the size of the return type and predicts the offset accordingly
	* Optionally, it can check if copy elision happened and if it did, verify that the predicted offset is correct

---
### Exercise
Implement the type returned by the `key()` member function. Create only skeleton class for the remaining functionality so that it would compile without taking too much work. You do not need to deal with any corner cases like initialising containers.

---
### Can it be done better?
The previous example has a big problem - all member classes have to be serialised (or all member classes that are not serialised need to be explicitly tagged as such).

Well, it can.

---
#### The power of garbage
When the `key()` method is called, it's possible to check out where was the last byte initialised before and to check later which was the first byte initialised after it.

This requires creating the class inside garbage. However, this isn't so easy:
```C++
CenteringAmplifier amplifier; // Members without initialisation are not initialised
CenteringAmplifier amplifier2(); // All members are initialised (memset to 0)
CenteringAmplifier amplifier3(config.amplifier); // All members are initialised
```

There is no way to create an object that takes arguments inside garbage. Furthermore, members can happen to be initialised to the same values as the garbage.

---
The parent class can use CRTP to know the child's size and fill it with garbage at the end of its constructor.

If the class is created twice, each time in a different garbage, it's possible to tell garbage from values that happen to the same value as garbage.

![Memory layout of multiple inheritance](../pictures/garbage_overwrite.png)

You can try the class analysing code online [here](https://repl.it/repls/UnusedImperturbableRam).

---
#### A place for stateful metaprogramming
This can be used to iterate through members more efficiently, but it does not allow naming or otherwise tagging the members.

A class can be aggregate initialised with types that have an universal conversion operator (let's assume it's called `ObjectInspector`).

```C++
// Partial template specialisation for recursively finding member count
template <typename T, typename sfinae, size_t... indexes>
struct MemberCounter {
  constexpr static size_t get() {
    return sizeof...(indexes) - 1;
  }
};

template <typename T, size_t... indexes>
struct MemberCounter<T, decltype( T {ObjectInspector<T, indexes>()...} )*, indexes...> {
  constexpr static size_t get() {
    return MemberCounter<T, T*, indexes..., sizeof...(indexes)>::get();
  }
};
```

---
We can then use stateful metaprogramming to store what types these actually converted to and use it to generate a set of conversion functions:
```C++
// Forward declares for ADL
template<typename T, int N> struct ObjectGetter {
 	friend void processMember(ObjectGetter<T, N>, T* instance, size_t offset);
  friend constexpr int memberSize(ObjectGetter<T, N>);
};

// The class that adds implementations according to its parametres
template<typename T, int N, typename Stored>
struct ObjectDataStorage {
  friend void processMember(ObjectGetter<T, N>, T* instance, size_t offset) {
    std::cout << N << ": " << *reinterpret_cast<Stored*>(reinterpret_cast<uint8_t*>(instance) + offset) << std::endl;
  };
  friend constexpr int memberSize(ObjectGetter<T, N>) {
    return sizeof(Stored);
  }
};

// The class whose conversions cause instantiations of ObjectDataStorage
template<typename T, int N>
struct ObjectInspector {
  template <typename Inspected, std::enable_if_t<sizeof(ObjectDataStorage<T, N, Inspected>) != -1>* = nullptr>
  operator Inspected() {
    return Inspected{};
  }
};
```

---
Finally, we can iterate through all those functions while predicting the layout (which can all be done at compile time).

This can only serialise objects as JSON arrays. The previous method can be used to insert names to the members, while using this method to improve performance and as a double check.

An implementation of this mechanism is [here](https://repl.it/repls/YellowgreenThriftyScientificcomputing).


---
## JSON-RPC
The use case is to have an abstraction that unifies access to local objects and remote objects. We want to have a class that can be used in two ways:
1. The class' methods access its internal attributes as usual, but can also be called through TCP (or UDP)
2. The class' methods actually call the methods of a different object accessed through TCP (or UDP)

The aim is to have a layer of abstraction that hides if the object is local or remote and to do it with as little code as possible.

JSON-RPC protocol assumes sending requests with method name and parametres and receives a reply with procedure name, response parametres and an optional error message. It can be used internally by this remote communication. The parametres can be of any type, but in this case, an array is the most suitable. We need to find a way to generate bindings for classes without too much code.

---
### OOP implementation
```C++
struct ILaserCutter : virtual IDevice {
	virtual void setPosition(float x, float y) = 0;
	virtual void setPower(float power_mw) = 0;
};

struct LaserCutterLocal : virtual public ILaserCutter, virtual public DeviceLocal {
	void setPosition(float x, float y) override;
	void setPower(float power_mw) override;
//...

struct LaserCutterRemote : virtual public ILaserCutter, virtual public DeviceRemote {
	void setPosition(float x, float y) override;
	void setPower(float power_mw) override;
//...
```
Methods in `LaserCutterRemote` can be one liners calling a generic function provided by a parent class of `DeviceRemote`. This isn't bad, but there are still many boring declarations to write.

---
### Unified implementation #1
```C++
class LaserCutter : public Device {
	void remoteMethods() override {
		Device::remoteMethods();
		method("setPosition", &LaserCutter::setPosition);
		method("setPower", &LaserCutter::setPower);
	}
public:
	void setPosition(float x, float y) {
		if (remote()) call("setPosition", x, y);
		// ...
	}
	void setPower(float power_mw) {
		if (remote()) call("setPower", power_mw);
		// ...
	}
}`
```
This no longer requires implementing two extra classes to create the abstraction, but there's still a lot of repetitive code. It would be handy to be able to swap methods dynamically.

---
### Unified implementation #2
```C++
class LaserCutter : public Device {
	void _setPosition(float x, float y) override;
	void _setPower(float power_mw) override;
public:
	RemoteMethod<&LaserCutter::_setPosition> setPosition = rpcMethod("setPosition");
	RemoteMethod<&LaserCutter::_setPower> setPower = rpcMethod("setPower");
//...
```
Using function pointers as template arguments without knowing the exact type in advance is a C++17 only feature:
```C++
template <auto Method>
class RemoteMethod //...
```

---
### Exercise
Implement the last iteration of the `Device` class. You don't have to write the methods that actually serialise and deserialise the arguments or send the information (it would take long), only what is necessary to get the `rpcMethod` method working.

---
## GUI
Can we use the pliability of C++ syntax to define a GUI with less code than the web frameworks?

Yes, we can.

---
```C++
struct NameWindow : Formulaire {
	Title t = title("Set address");
	struct : HBox {
		Input<std::string> first = placeholderText("First name");
		Input<std::string> last = placeholderText("Last name");
	} name = {{ noBorder().title("Name") }};
	Input<std::string> address = title("Address");
	struct : HBox {
		Input<std::string> city = placeholderText("City").defaultValue("Brno");
		Input<int> code = placeholderText("Postal code");
	} city = {{ noBorder().title("City") }};
	Button submit = title("Submit");
};
//...
wnd.submit = [=] () {
	addEntry(*wnd.name.first + " " + *wnd.name.last,
		*wnd.address + ", " + std::to_string(*wnd.city.code)
		+ " " + wnd.city.city);
	wnd.close();
};
```

---
The source code is at: [https://github.com/Dugy/DuGUI](https://github.com/Dugy/DuGUI)

So now, how it works?

---
* The property setting member functions create objects containing settings that also have pointers to the parent object
* The individual widgets (`Input`, `Button`, ...) receive the settings and the pointer to their parent object and add themselves to the list of its children
* The containers are initialised in the same way, but if they are instanced at the same place as they are defined, they can be only initialised using aggregate initialisers
* The aggregate initialisers expect the parent class' constructor arguments before member initialisers, so the initialising object is passed to the parent
* Together, it creates a tree structure at runtime

---
## Homework
Design and implement a small library for parsing command line arguments that is as easy to use and needs as little code as possible.
```
./railgun enemies.csv -tbf --auto-aim --multitarget 3 -p priority_rules.conf
```

---
## Conclusion
If you have gone this far without skipping exercises and homework, go buy a cloak and a pointed hat and you are done.
