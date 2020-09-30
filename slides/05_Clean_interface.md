# 5. Clean Interface
There are some more ways that can be used to reduce the volume of code and they aren't even particularly complicated. They are, however, often forgotten.

---
## Bit flags
This is one of the oldest ones, it's older than C++ itself.

There is a function, set of functions or even template class that has a lot of options that slightly alter the execution, but default values are usually good enough and they are mostly not arithmetic. A fairly common occurrence. It may be solved using a lot of optional arguments, a settings object or using bit flags. Implementing some of the more complex cases requires some knowledge of bit arithmetic, but using it is straightforward.

---
```C++
struct PermissionFlags {
	enum Flags {
		NONE = 0;
		OWNER_READ = 1 << 0,
		OWNER_WRITE = 1 << 1,
		OWNER_EXECUTE = 1 << 2,
		GROUP_READ = 1 << 3,
		GROUP_WRITE = 1 << 4,
		GROUP_EXECUTE = 1 << 5,
		OTHERS_READ = 1 << 6,
		OTHERS_WRITE = 1 << 7,
		OTHERS_EXECUTE = 1 << 8,
	};
};
void setFileFlags(const std::string& fileName, int flags = PermissionFlags:NONE) {
	if (flags & OWNER_READ) {
//...
setFileFlags("superFile.png", PermissionFlags::OWNER_READ | PermissionFlag::OWNER_WRITE);
```
`enum class` cannot be used here, but toplevel constants can be avoided anyway.

---
With a bit of bit arithmetic, even numbers can be passed as optional arguments.
```C++
struct ParseFlags {
	constexpr static int WIDTH_OFFSET = 8;
	constexpr static int WIDTH_MASK = 0xff << 8;
	enum Flags {
		COMMA_SEPARATOR = 1 << 0,
		SEMICOLON_SEPARATOR = 1 << 1,
		SPACE_SEPARATOR = 1 << 2,
		TAB_SEPARATOR = 1 << 3,
		DECIMAL_IS_COMMA = 1 << 4,
		PRESET = (2 << WIDTH_OFFSET) | TAB_SEPARATOR,
	};
	static Flags width(int set) {
		return Flags(set << WIDTH_OFFSET);
	}
};
std::vector<std::vector<float>> parseCsv(const std::string& source, int flags = ParseFlags::PRESET) {
	if ((flags & ParseFlags::COMMA_SEPARATOR) && (flags & ParseFlags::DECIMAL_IS_COMMA))
		throw std::logic_error("Comma can't be both a separator and a decimal");
//...
auto data = parseCsv("experimentalData.csv", ParseFlags::SPACE_SEPARATOR | ParseFlags::width(3));
```

---
### Exercise
Implement the `parseCsv()` function, but add also flags for throwing with any mistake in file (otherwise it tries to go through), for throwing when the file is missing (otherwise it returns an vector) and a unified one for throwing in case of any of the earlier two mistakes (and do this one with only a single line of additional code).

```C++
auto data = parseCsv("experimentalData.csv", ParseFlags::PEDANTIC | ParseFlags::width(4));
```

Here's an example of code that can parse csv files:
```C++
std::vector<std::vector<float>> parsed;
std::string line;
while (std::getline(input, line, '\n')) {
	parsed.emplace_back();
	auto& writing = parsed.back();
	std::string number;
	std::stringstream lineStream(line);
	while (std::getline(lineStream, number, ',')) {
		writing.push_back(std::stof(number));
	}
}
```

---
## Function overloading
If two functions do the same thing, it makes sense to give them identical names.
```C++
void rotate(Angle pitch, Angle roll, Angle yaw);
void rotate(Angle rotation, Vector3 axis);
```

Constructor overloads are often hard to avoid, because copy constructor and move constructor are necessarily overloads of some other constructor.

It's also possible to have multiple overloads for operator overloads.

---
### Utility in generic programming
Function overloading really shines when using generic functions.
```C++
template <typename T>
void save(const std::string& fileName, T value) {
	std::ofstream file(fileName);
	file << value; // one of the dozens of overloads could match...
}
```

---
### Problems
What happens if the actual type matches more overloads? These are simplified rules about priority:
* Exactly the same type is chosen first
* A template that can be specialised to the type goes after
* A type that can be converted to the right one has low priority

If these rules cannot be used to decide what has priority, the program is invalid because of ambiguous overload. This can be hard to deal with if there's a lot of implicit conversions available, default arguments, combinations of templates and conversions etc. MSVC makes it even worse by not abiding to the rules. Such problems can be solved using SFINAE only or `if constexpr` copiously.

---
### Exercise
Create a set of overloaded functions that save variables to files. It must be able to save these types in the following ways:
* `std::string` is saved normally
* numeric types (identified via SFINAE) are saved as strings that are output by `std::ostream`'s `operator<<`
* pointers to these types are dereferenced and the contents are saved
* same for shared and unique pointers

---
## Friend methods
The usual use of `friend` declaration is to grant access to private and protected members to a specific class or function. It's useful if a group of classes is supposed to work together, but without restricting their access to the limitations we need to impose to unrelated classes.

You probably know this.

---
It's also useful to write pseudo-methods that need to use multiple instances of the class (it's actually the intended way of doing so).
```C++
class Millimetres {
	float value;
	public:
	// ...
	friend Vector3<Millimetres> vectorise(Milimetres x, Milimetres y, Milimetres z) {
		return Vector3<Millimetres>{ x, y, z};
	}
};
//...
auto position = vectorise(parse("x"), parseMm("y"), parseMm("z"));
```
The `vectorise` function looks like a free function, but it isn't. Its existence isn't considered if neither of the arguments is of the `Millimetre` class.

---
It is also useful to conveniently overload operators from the right side:
```C++
struct Millimetres {
	float value;
	friend std::ostream& operator<<(std::ostream& out, const Millimetres& val) {
		return out << val.value << " mm";
	}
};
```

---
If the friend function's arguments don't contain the type where it's declared, it must be converted into a free function using a forward declaration:
```C++
class Block {
	float value;
	// ...
	public:
	//...
	friend void decrease(BlockAccessor& accessor) {
		auto lock = accessor.lock();
		Block& block = lock->block;
		block.value--;
	}
};
void decrease(BlockAccessor&); // can be also backwards-declared
```

---
## Auxiliary objects
It may be useful to create objects for the sole purpose of storing some settings so that they would not have to be written again and again.
```C++
auto remote = makeRemote("turboencabulator");
remote->addMethod("initialise", this, &Turboencabulator::initialise);
{
	auto basePlate = remote->makeComponent("basePlate");
	basePlate->addMethod("temperature", this, &Turboencabulator::basePlateTemperature);
	basePlate->addMethod("load", this, &Turboencabulator::basePlateLoad);
	//...
```

---
### Auxiliary objects with RAII
An auxiliary object's destructor is called when the object runs out of scope, so it can be used to send a prepared message, commit a change or check if all fields were completed.

```C++
{
	Command cmd(communicator, "destroy");
	cmd->*"target" = "PQ712";
	cmd->*"protocol" = "Cobra";
	cmd->*"strategy" = "B27";
} // sent
```
Be careful about errors. They need to be handled with exceptions and destructors that throw need to be marked as such or the program will terminate. Also, throwing destructors shouldn't be used for anything else than fatal erros because an exception thrown during exception handling will cause the program to abort if it reaches the context where the first exception is handled.

---
## Singleton
Singleton is a class that can have only one instance. This allows the instance to be accessible from anywhere, helping avoid the necessity to pass references all around the program.
```C++
Logger::getInstance().addDataPoint("heater voltage", voltage);
```

It is also a highly controversial pattern that will spaghettify your code if used liberally. There must be a strong motivation to turn a class into a singleton.

---
```C++
class Input final {
	Input() {
		// ...
	}
	Input(const Input&) = delete;
	Input(Input&&) = delete;
public:
	static Input& getInstance() {
		static Input instance;
		return instance;
	}
	//...
}
```

---
### The dangers
* Modifying the program to have more instances of the class requires extensive changes
* A singleton may be used an inappropriate layers of abstraction, tangling all dependencies and causing a huge mess
* The order of destruction of singletons is not defined and they are initialised in the order they are accessed
* A singleton is difficult to test automatically, because the tests cannot destroy it and create it anew
* If a singleton has a state, it can be left in an unsuitable one at a location that is difficult to find
* Reaching a singleton in a debugger is difficult

---
### How not to shoot yourself in the foot
* Use a singleton only if you are sure there will never be any need for more instances (user input, GUI set, video card access, master server connection)
* Make sure that the singleton encapsulates some functionality entirely and will never be needed by anything it uses
* Ensure that using the singleton won't be able to cause damage if used on a wrong layer of abstraction or make it unavailable there
* If the singleton' depends on other singletons, have `main()` or some other privileged function create it and destroy it explicitly
* If automatic testing is applicable, implement some way to reset it
* If the singleton is to have a state, it should be `thread_local` or mutex-protected and RAII should guarantee it will not be left in a bad one

---
### A safer and testing-friendly singleton
```C++
class Input final {
	Input() {
		// ...
	}
	Input(const Input&) = delete;
	Input(Input&&) = delete;
	static std::unique_ptr<Input>& getInstanceInternal() {
		static std::unique_ptr<Input> instancePtr;
		return instancePtr;
	}
	static void setInstance(std::unique_ptr<Input> assigned) {
		auto& instance = getInstanceInternal();
		instance = std::move(assigned);
	}
	static void terminate() {
		auto& instance = getInstanceInternal();
		instance.reset();
	}
	friend int main(int, char**); // privileges some parts to safely create it or destroy it
	friend class Test; // allows tests to reset it
public:
	static Input& getInstance() {
		return *getInstanceInternal();
	}
	//...
};
```

---
## Enum inheritance
It happens that there is an `enum` type that needs to be extended by other classes. It can be faked by combining classes and plain enums:
```C++
struct LayoutType {
	enum Type {
		HBox,
		VBox,
		Max
	};
};
struct WidgetType : public LayoutType {
	enum Type {
		LineEdit = LayoutType::Max,
		CheckBox,
		Max,
	};
};
```

A disadvantage is that if it's accepted as an argument to a function, it has to be implicitly cast to `int`.

---
## Aggregate initialisation
This feature is mainly a C remnant intended to keep C code compatible with C++, but it can be useful to avoid having to write constructors.

Requirements for the class to be aggregate initialisable (as of C++17):
* Can't be polymorphic (virtual methods are forbidden)
* Can't have private or protected parent classes or member variables (parent classes themselves can have them)
* Can't have custom defined constructors

---
```C++
struct Parent {
  int a;
  Parent(int a) : a(a) {}
};

struct Child : Parent {
  int b = 3;
  int c;
  int d;
};
//...
Child child = {{3}, 2, 4};
```

Parent classes are initialised through included brace-enclosed lists (can call constructors or be aggregate-initialised as well). Default values are overriden. Supernumerary members are not initialised.

---
For clarification, member names can be written explicitly:
```C++
struct Octopus : Animal {
	int tentacles;
	bool changesColour;
};zle
//...
Octopus octopus = {{basicProperties}, .tentacles = 10, .changesColour = true};
```

---
Aggregate initialisation can be used with dynamic allocation, but not with smart pointers. This restricts its usefulness.

This was used much more in C, C++ has constructors that are more convenient to use. However, if the class is simple and created at only one location (or maybe two), writing a constructor can be unnecessarily lengthy.

```C++
struct { // Voldemort type, type name cannot be written
	Element* parent;
	int x;
	int parentOffset() const {
		return parent->parent->offset.x;
	}
} offset = { this, 0 };
```

---
## Design philosophy
At this point, you should know a load of tricks how to write less code when using a lower layer of your program. Now it's time to sit down and reflect about their usage.

Every approach has some tradeoffs between amount of code needed for using it, amount and complexity of code to get it working, performance and error proneness.

---
When deciding whether to design an interface that needs little code:
* Is the code used often enough? If not, keeping up with SOLID principles may be enough
* Can performance matter in that part of code? If yes, don't use tricks that rely on virtual functions or dynamic allocation (`std::function` uses virtual functions and will resort to dynamic allocation for larger functors) or other performance-unfriendly tricks

---
When looking for a way to allow using something with minimal code, it's better not to think about SOLID principles, but from the point of view of the user of the interface:
* Every piece of information has to be written at least once
* Some parts of the code have to be written in any case, like types of members
* Anything else usually doesn't need to be written over and over

The following code shows a getter/setter pair that convert a value and call a protocol need only one line to be wholly functional:
```C++
class Motor : RemoteDevice {
	const std::function<void(int, int, int)> callibrate = remoteMethod("cal");
	remoteValue<float> positionX = access("xpos");
	remoteValue<float> positionY = access("ypos") & ratio(0.15);
//...
callibrate(20, 10000, 300);
positionX = 10;
positionY = 12;
```

---
* Template parameter deduction can be used to avoid writing types more than once
* Parent classes help avoid writing similar implementations to interfaces and paths to member functions
* Function argument values can be stored
* Method calls can often be shortened to operators

---
When deciding if an implementation that can be used with less code should be used over one that uses more code:
* Is the idea too different from typical C++ code? If yes, is it used often enough to justify that every new programmer will have to read the documentation?
* How easy is it to make mistakes by being careless? Can these mistakes be detected before they cause undefined behaviour?
* Sometimes, it's better to need slightly more code if it can prevent wrong usage on syntactic level

---
## Homework
Implement the functionality surrounding the `remoteValue` and `remoteMethod` functions or methods.
