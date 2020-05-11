# 5. Clean Interface
There are some more ways that can be used to reduce the volume of code and they aren't even particularly complicated. They are, however, often forgotten.

---
## Bit flags
This is one of the oldest ones, it's older than C++ itself.

There is a function, set of functions or even template class that has a lot of options that slightly alter the execution, but default values are usually good enough and they are mostly not arithmetic. A fairly common occurrence. It may be solved using a lot of optional arguments, a settings object or using bit flags. Implementing some of the more complex cases requires some knowledge of bit arithmetic, but using it is straightforward.

---
```C++
struct PermissionFlags
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
## Return value overloading
This doesn't work:
```C++
// Outputs a shared_ptr to a valid default-initialised object
template <typename T>
std::shared_ptr<T> defaultShared();
```

But it could be very useful to shorten code:
```C++
namePtr(std::make_shared<std::string>()),
// becomes
namePtr(defaultShared());
```

But is there really no trick to get it running anyway?

---
There is one! We can abuse implicit conversion.

There is a non-template function that returns a single type. The type can be implicitly converted to whatever we want. Conversion constructor cannot be used because `std::shared_ptr` is standardised, but the returned class may have a conversion operator defined.

```C++
struct SmartPointerInitialiser {
	template <typename T>
	operator std::shared_ptr<T>() {
		return std::make_shared<T>();
	}
	template <typename T>
	operator std::unique_ptr<T>() {
		return std::make_unique<T>();
	}
};
SmartPointerInitialiser defaultSmart() {
	return SmartPointerInitialiser{};
}

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
	Command cmd("destroy");
	cmd->*"target" = "PQ712";
	cmd->*"protocol" = "Cobra";
	cmd->*"strategy" = "B27";
} // sent
```
Be careful about errors. They need to be handled with exceptions and destructors that throw need to be marked as such or the program will terminate.

---
## Stateful initialisation
In most cases, the program is easier to use if it's stateless. It cannot be detected at compile time that an object is in a state where some operation cannot be executed on it, making it error-prone and counter-intuitive.

But in some cases, it can be useful. In some cases, auxiliary objects with RAII can be used to ensure the state is reset when done.
```C++
auto state = makeRemote("turboencabulator");
addMethod("initialise", this, &Turboencabulator::initialise);
{
	auto state = makeComponent("basePlate");
	addMethod("temperature", this, &Turboencabulator::basePlateTemperature);
	addMethod("load", this, &Turboencabulator::basePlateLoad);
	//...
```

The state may be stored in a parent method. But it can also be in a `static thread_local` variable.

---
## Singleton
Singleton is a class that can have only one instance. This allows the instance to be accessible from anywhere, helping avoid the necessity to pass references all around the program.
```C++
auto response = ServerConnection::getInstance().sendMessage(command);
```

It is also a highly controversial pattern that will spaghettify your code if used liberally.

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
* A singleton may use other classes, some of which may be later altered to use the singleton, tangling all dependencies and causing huge mess
* The order of destruction of singletons is not defined and they are initialised in the order they are accessed
* A singleton is difficult to test automatically, because the tests cannot destroy it and create it anew
* If a singleton has a state, it can be left it an unsuitable one at a location that is difficult to find
* Reaching a singleton in a debugger is difficult

---
### How not to shoot yourself in the foot
* Use a singleton only if you are sure there will never be any need for more instances (user input, GUI set, video card access, server connection)
* Make sure that the singleton encapsulates some functionality entirely will never be needed by anything it uses
* If the singleton' depends on other singletons, have `main()` or some other privileged function create it and destroy it explicitly
* If automatic testing is applicable, implement some way to reset it
* If the singleton is to have a state, it should be `thread_local` or mutex-protected and RAII should guarantee it will not be left in a bad one

---
### A safer singleton
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
	static void initialise() {
		auto& instance = getInstanceInternal();
		instance = std::unique_ptr<Input>(new Input);
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
## Design
At this point, you should know a load of tricks how to write less code. Now it's time to sit down and reflect about their usage.

---
## Homework
Implement the functionality surrounding the `makeRemote` and `addMethod` functions or methods.