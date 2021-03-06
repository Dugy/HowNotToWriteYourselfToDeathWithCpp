# 1. Introduction: Don't repeat yourself
_I know you probably know this, but some introduction is needed, right?_

Don't Repeat Yourself (DRY). More importantly, do your best to avoid having to repeat yourself in the future. And others having to repeat themselves.

---
## The consequences of repeating yourself
The opposite of DRY is We Enjoy Typing or Write Everything Twice. The WET principles are the root of all pain.

---
### More work
You can use copy-paste. In this case, every single line has to be edited:
```C++
lenses[0].current = 0;
lenses[1].current = 0;
lenses[2].current = 0;
lenses[3].current = 0;
lenses[4].current = 0;
lenses[5].current = 0;
lenses[6].current = 0;
lenses[7].current = 0;
// ...
```

---
### More work in the future
That was just typing a few numbers, eh? Here it gets more annoying:
```C++
std::string id;
if (entry->has_child("id"))
  id = entry->get_child("id").contents();
std::string name;
if (entry->has_child("name"))
  name = entry->get_child("name").contents();
std::string description;
if (entry->has_child("description"))
  description = entry->get_child("descirption").contents();
// ...
```
Bonus, how do you change it to `nlohmann::json`? If you are smart, you might succeed with a bunch of Find&Replace... supposing the code is this simple.

---
### Error prone
The code will crash because of the typo that's unlikely to ever be spotted.
```C++
std::string description;
if (entry->has_child("description"))
  description = entry->get_child("descirption").contents();
```
It's also very easy to forget to paste the `"description"` string over every occurrence of the string that was pasted, leaving the old one somewhere.

---
### Atrocious to use
```C++
XML::Element* idElem = new XML::Element("id");
idElem->setContents(saved.id);
retval->addElement(idElem);
XML::Element* nameElem = new XML::Element("name");
nameElem->setContents(saved.name);
retval->addElement(nameElem);
XML::Element* descriptionElem = new XML::Element("description");
descriptionElem->setContents(saved.description);
retval->addElement(descriptionElem);
// ...
```
Now, if you want to add a new property named `tag`, you have to add it both to the loading section and saving section, and maybe to several additional places, so you'll forget one easily.

---
### Unreadable
```C++
XML::Element* descriptionElem = new XML::Element("description");
descriptionElem->setContents(saved.description);
retval->addElement(descriptionElem);
XML::Element* briefElem = new XML::Element("brief");
briefElem->setContents(saved.brief);
retval->addElement(briefElem);
XML::Element* versionElem = new XML::Element("version");
versionElem->setContents("v" + saved.version);
retval->addElement(versionElem);
XML::Element* authorElem = new XML::Element("author");
authorElem->setContents(saved.author);
retval->addElement(authorElem);
```
How easy is it to notice that `version` is serialised slighly differently?

---
## Technical debt
When changing code that is already written, a need for duplicating a part of code may arise. It might be slightly edited and used for a different functionality, in some `if` block or something.
```C++
std::string id;
if (format == Format::XML) {
  if (entry->has_child("id"))
    id = entry->get_child("id").contents();
} else if (format == Format::JSON) {
  if (json.count("id") > 0)
    id = json["id"];
}
```
While these lines can be written quickly, it will slow further development down. Later additions to the deserialisation code will have to be duplicated as well.

Have you noticed that _four_ edits are needed to rename the property?

---
This problem is called *technical debt*. If you don't repay it, it will grow indefinitely, until development is nigh impossible.

A single duplication is not a big deal and fixing it may not even be worth the effort. A fivefold repetition of similar code is a budding problem. Having to repeat some already repetitive code twice any time some tool is used is a clear sign that tool is used badly or badly designed.

Having to write some repetitive piece of code more than once is a sign of impending disaster.

---
When some code is repeated, a tool for doing it without writing too much code is needed. The more it is used, the more complex it can be (in that case, automated tests are needed). Not having to write repetitive code when using it can justify using repetitive code when implementing it.
```C++
class ComponentInformation : public Serialisable {
  int id = key("id") = -1;
  std::string name = key("name") = "Unnamed";
  Version version = key("version") = "1.0.0";
  // ...
};
```
Because there are so many reasons why the code above cannot work, its implementation will be explained in the last lesson. Let us start first with the easiest methods.

---
## Requirements
* Experience with programming
* Knowledge of C++ sufficient to write common stuff without checking solutions online
* Understanding of object oriented design

Being able to read C++ code and having used some other object-oriented programming language is **not enough.**

---
## Coverage
Stuff we will learn:
* How to achieve the same functionality with less code
* How to take your time to accelerate a lot of future development
  * Less code in the future
  * Less future errors to deal with
* A lot of C++ tricks to achieve unusual functionality

Stuff that will **not** be covered:
* Advanced algorithms
* Optimising code for speed
* Code style

---
Note: sometimes, faster to write code can be faster, but there will also be performance hindering consequences of writing shorter code. In those cases, you will have to consider if they're not _premature pessimisation_ (slower code for too little or no benefit).

Note #2: Knowing the general guidelines for writing good code often isn't enough. The rationale behind them must be properly understood. Not doing so leads to _cargo cult programming,_ adhering to rules religiously rather than rationally.

---
## Outline
1. Introduction
2. Object model in C++
3. Nontrivial operator usage
4. Generic functions and SFINAE
5. Clean interface
6. Generic classes
7. Memory manipulation
8. Runtime metaprogramming
9. Template metaprogramming
10. Putting it all together

---
## Common ways of repetitive code removal

Often, all that's needed is the will to write the code without technical debt.
```C++
lenses[0].current = 0;
lenses[1].current = 0;
lenses[2].current = 0;
lenses[3].current = 0;
lenses[4].current = 0;
lenses[5].current = 0;
lenses[6].current = 0;
lenses[7].current = 0;
// ...
```

```C++
for (auto& lens : lenses)
  lens.current = 0;
```

---
### Just not being as lazy as a toad
```C++
if (lines[i].find("<trait>") != std::string::npos)
    in_trait = true;
else if (lines[i].find("</trait>") != std::string::npos)
    in_trait = false;
else if (lines[i].find("<object>") != std::string::npos)
    in_object = true;
else if (lines[i].find("</object>") != std::string::npos)
    in_object = false;
else if (lines[i].find("<stage>") != std::string::npos)
    in_stage = true;
else if (lines[i].find("</stage>") != std::string::npos)
    in_stage = false;
else if (lines[i].find("<cfg>") != std::string::npos)
    in_cfg = true;
else if (lines[i].find("</cfg>") != std::string::npos)
    in_cfg = false;
else if (lines[i].find("<goal>") != std::string::npos)
    in_goal = true;
else if (lines[i].find("</goal>") != std::string::npos)
    in_goal = false;
// 150 more lines of this
```
Note: I stumbled upon this code in a real project, though it was originally written in Python.

---
Using regex instead of substring search:
```C++
  static const std::regex regex("\\<([_/a-z]+)\\>");
  std::smatch match;
  if (std::regex_match(line, match, regex)) {
    if (match.size() == 2) {
        std::ssub_match subMatch = match[1];
        std::string found = subMatch.str();
        if (found[0] != '/')
          blocksWeAreIn.insert(found);
        else {
          auto name = blocksWeAreIn.find(found.substr(1));
          blocksWeAreIn.erase(name);
        }
    }
  }
```
The code also runs faster.

---
Even that regex isn't so necessary:
```C++
std::unordered_set<std::string> blocksWeAreIn;
const std::string& line = lines[i];
for (size_t i = 0; i < line.size(); i++) {
  if (line[i] == '<') {
    i++;
    std::string blockName;
    while (line[i] != '>' && line[i]) {
      blockName.push_back(line[i]);
      i++;
    }
    if (blockName[0] != '/')
      blocksWeAreIn.insert(blockName);
    else {
      auto found = blocksWeAreIn.find(blockName.substr(1));
      blocksWeAreIn.erase(found);
    }
  }
}
```

---
### Loops
```C++
analogWrite(LED[0], brightness);
delay(lightTime);
analogWrite(LED[0], 0);
delay(lightTime);
analogWrite(LED[1], brightness);
delay(lightTime);
analogWrite(LED[1], 0);
delay(lightTime);
analogWrite(LED[3], brightness);
delay(lightTime);
analogWrite(LED[3], 0);
delay(lightTime);
analogWrite(LED[6], brightness);
delay(lightTime);
analogWrite(LED[6], 0);
delay(lightTime);
analogWrite(LED[8], brightness);
delay(lightTime);
// More LEDS to blink
```
The sequence of LEDs is 0, 1, 3, 6, 8, 7, 5, 2, so a `for` cycle may seem somewhat unusable....

---
But it's too bad without a cycle. A quick thought is enough to figure out how to use one:
```C++
for (int i : {0, 1, 3, 6, 8, 7, 5, 2}) {
  analogWrite(LED[i], brightness);
  delay(lightTime);
  analogWrite(LED[i], 0);
  delay(lightTime);
}
```

However, you should not exaggerate it:
```C++
for (int i : {0, 1, 3, 6, 8, 7, 5, 2}) {
  for (int j = brightness; j >= 0; j -= brightness) {
    analogWrite(LED[i], j);
    delay(lightTime);
  }
}
```

---
### Lambda functions
How can we improve this?
```C++
XML::Element* descriptionElem = new XML::Element("description");
descriptionElem->setContents(saved.description);
retval->addElement(descriptionElem);
XML::Element* briefElem = new XML::Element("brief");
briefElem->setContents(saved.brief);
retval->addElement(briefElem);
XML::Element* versionElem = new XML::Element("version");
versionElem->setContents(saved.version);
retval->addElement(versionElem);
XML::Element* authorElem = new XML::Element("author");
authorElem->setContents(saved.author);
retval->addElement(authorElem);
```

---
Because the trick mentioned earlier is not very convenient, we can improve it using lambdas:
```C++
auto serialiseString = [&retval] (const std::string& key, const std::string& serialised) {
  XML::Element* element = new XML::Element(key);
  element->setContents(serialised);
  retval->addElement(element);
};
serialiseString("description", saved.description);
serialiseString("brief", saved.brief);
serialiseString("version", saved.version);
serialiseString("author", saved.author);
```
Of course, it doesn't have to be a lambda. Depending on the situation, it might be better to use a helper function, a helper class, a helper method, a parent class with helper methods et cetera.

---
#### Generic lambdas
Now, the same, but types are different:
```C++
XML::Element* descriptionElem = new XML::Element("description");
descriptionElem->setContents(saved.description);
retval->addElement(descriptionElem);
XML::Element* idElem = new XML::Element("id");
idElem->setContents(std::to_string(saved.id)); // integer
retval->addElement(briefElem);
XML::Element* ratingElem = new XML::Element("rating");
versionElem->setContents(std::to_string(saved.rating)); // float
retval->addElement(ratingElem);
```

---
If some code works for more types (for example using function overloading), one of the lambda arguments can be declared as `auto` and used for various types:
```C++
auto serialise = [&retval] (const std::string& key, auto serialised) {
  XML::Element* element = new XML::Element(key);
  std::stringstream stream;
  stream << serialised;
  element->setContents(stream.str());
  retval->addElement(element);
};
serialise("description", saved.description);
serialise("id", saved.id);
serialise("rating", saved.rating);
```

Note: Parametres and return values can be declared as `auto` also in inline functions. It isn't possible to get a pointer to any function with an `auto` as parameter nor to wrap it into `std::function`.

---
### Exercise
Improve this code:
```C++
display(3,0);
if (n<10) { display(4,0); display(5,n);}
else if (n<20) { n = n-10; display(4,1); display(5,n);}
else if (n<30) { n = n-20; display(4,2); display(5,n);}
else if (n<40) { n = n-30; display(4,3); display(5,n);}
else if (n<50) { n = n-40; display(4,4); display(5,n);}
else if (n<60) { n = n-50; display(4,5); display(5,n);}
else if (n<70) { n = n-60; display(4,6); display(5,n);}
else if (n<80) { n = n-70; display(4,7); display(5,n);}
else if (n<90) { n = n-80; display(4,8); display(5,n);}
else if (n>=90) { n = n-90; display(4,9); display(5,n);}
```

The value of `n` will be in the range from 0 to 99.

---
Improve this code:
```C++
delay(50);
clear(i);
NumberZero(i);
delay(50);
clear(i);
NumberOne(i);
delay(50);
clear(i);
NumberTwo(i);
delay(50);
clear(i);
NumberThree(i);
delay(50);
clear(i);
NumberFour(i);
delay(50);
clear(i);
NumberFive(i);
delay(50);
```
`NumberZero`, `NumberOne` et cetera are free functions that take an int argument and return void

---
### Exceptions
Don't shun exceptions.
```C++
Result<PowerSystem> getOutputVoltage(double& voltage) const {
  Result<BV13> result = _generator->getOutputVoltage(voltage);
  return {
    result.summary(),
    (result.code() == BV13::Results::Ok) ? Results::Ok : Results::HardwareError,
    result.brief(),
    result.details(),
    result.notes()
  };
}
```
```C++
double getOutputVoltage() const {
  return _generator->getOutputVoltage();
}
```
Avoid unnecessary error codes like a plague, for they will spread.

---
Exceptions are not your enemy.
```C++
Result<PowerManagement> getAverageVoltage(double& voltage) const {
  double total = 0;
  for (auto& system : _powerSystems) {
    double voltageThere;
    auto result = system->getOutputVoltage(voltageThere);
    if (result.code() != PowerSystem::Results::Ok)
      return Result<PowerManagement>{ "A system failed", Results::HardwareFailure, "Fail", "", "" };
    total += voltageThere;
  }
  voltage = total / _powerSystems.size();
  return Result<PowerManagement>{ "Success", Results::Ok, "Ok", "", "" };
}
```
```C++
double getAverageVoltage() const {
  double total = 0;
  for (auto& system : _powerSystems)
    total += system->getOutputVoltage();
  return total / _powerSystems.size();
}
```

---
Without exceptions:
* The logic is much more complex
* Additional operations have to be done at runtime even if everything is okay
* Details about problems are often lost
* Failures often pass unnoticed

With exceptions:
* Far less code
* More intuitive code (return value is the value we wanted to get)
* No runtime overhead if everything is okay
* All problems can be appropriately reported in a central hub or caught by a debugger

---
Downsides of exceptions:
* If one is raised, the cost is high
* Implementation is compiler-dependent and messy
* Functions may exit at counter-intuitive locations, causing resource leaks if resources don't have proper destructors

---
Use exceptions when:
* There is a chance of failure, unlikely enough to need special treatment but likely enough to have consequences if ignored
* If a programming error elsewhere would cause hard-to-debug undefined behaviour
* To deal with incorrect use of the program

_Don't_ use exceptions when:
* If it is part of a use case
* If it is thrown too often in the context
* When the program is about to be released and too many contexts lack catch blocks

Use them as often as possible without making it annoying to run the program in a debugger set to pause on all exceptions.

---
And when using exceptions, don't abuse the possibility to throw any type:
* All exceptions should inherit from `std::exception`
* If the error happens due to external circumstances, throw `std::runtime_error` or better, an exceptions that inherits from it
* If the error happens due to programming errors, throw `std::logic_error`

This way, you won't have to resort to using `catch(...)`, which doesn't allow reporting what actually happened. The overhead of constructing exceptions and analysing them is neglectful compared to the overhead of throwing them.

```C++
struct PowerError : std::runtime_error {
  using std::runtime_error::runtime_error;
};
```
### Exercise
Improve this code:
```C++
JSON::Entry* idEntry;
if (!json->getEntry("id", idEntry))
  return false;
if (!idEntry->getAsType(_id))
  return false;
JSON::Entry* nameEntry;
if (!json->getEntry("name", nameEntry))
  return false;
if (!nameEntry->getAsType(_name))
  return false;
JSON::Entry* portEntry;
if (!json->getEntry("port", portEntry))
  return false;
if (!portEntry->getAsType(_port))
  return false;
```
Challenge: Do it without exceptions, but without macros or more than one line per entry.

---
## Avoiding error prone constructs
Searching for an error through many files can take significantly more time than writing a few lines of code. So making some code less prone to errors can be more important than low line count. That's basically the only purpose of const correctness and encapsulation.

Other constructs that exist mainly to help avoid errors are smart pointers, `std::vector::at` or `std::jthread`.

---
So, for example writing the last line is very easy to forget and few would notice if it was accidentally removed.
```C++
XML::Element* parsed = XML::parseFile("settings.xml");
Settings* settings;
try {
  settings = new Settings(parsed);
} catch (...) {
  std::cerr << "Loading settings failed!";
}
if (!settings) {
  settings = new Settings();
}
//...
delete parsed;
```
Also, `settings` isn't initialised to null pointer, so failing to load the settings would often cause a segfault instead of a generation of default settings.

---
It's better with unique pointers:
```C++
std::unique_ptr<Settings> settings;
try {
  std::unique_ptr<XML::Element> parsed = XML::parseFile("settings.xml");
  settings = std::make_unique<Settings>(parsed);
} catch (...) {
  std::cerr << "Loading settings failed!";
}
if (!settings) {
  settings = std::make_unique<Settings>();
}
```
This needs somewhat more letters, but there is much less space to make errors. It is possible to use a macro to rename `std::make_unique` to `mku` or something, but it can worsen readability.

---
### Self-registering Factory
We have an abstract class named `PeripheryDriver` that has three specific classes, `Voltmeter`, `Ampermeter` and `Wattmeter`. If they all abide by Liskov Substitution Principle, they all differ only in the way they are constructed.

The usual solution is to write a factory:
```C++
std::unique_ptr<PeripheryDriver> makeDriver(const std::string& type, const BusPosition& position) {
  if (type == "Voltmeter")
    return std::make_unique<Voltmeter>(position);
  if (type == "Ampermeter")
    return std::make_unique<Ampermeter>(position);
  if (type == "Wattmeter")
    return std::make_unique<Wattmeter>(position);
}
```
There is some repetition, but the `"Voltmeter"` is not the same as `Voltmeter`. The problem is that if someone creates an `Oscilloscope` class, how do we prevent him from forgetting to add it to this list that is in a completely unrelated file?

---
The solution is the self-registering factory:
```C++
std::unique_ptr<PeripheryDriver> makeDriver(const std::string& type,
                                          const BusPosition& position) {
  return driverGenerators[type](position);
}
```
And this is at the end of the file of each of the devices:
```C++
bool registerDriver() {
  driverGenerators["Voltmeter"] = [] (const BusPosition& position) {
    return std::unique_ptr<Voltmeter>();
  };
  return true;
}
const bool registered = registerDriver();
```
Initialising the file-scoped global variable (happens before main) registers the driver into the global map used by the factory.

---
Note: while this uses an unprotected global variable, it can be converted into a properly encapsulated singleton. The singleton and the factory themselves can be generic, allowing to write only one factory ever, but generics will be discussed later.

---
## Homework
Improve this code:
```C++
#define none std::string::npos
if (switches.find("-h") != none || switches.find("--help") != none)
  runProperties[PROP_HELP] = true;
if (switches.find("-v") != none || switches.find("--verbose") != none)
  runProperties[PROP_VERBOSE] = true;
if (switches.find("-r") != none || switches.find("--recursive") != none)
  runProperties[PROP_RECURSIVE] = true;
if (switches.find("-f") != none || switches.find("--force") != none)
  runProperties[PROP_FORCE] = true;
if (switches.find("-F") != none || switches.find("--find-missing") != none)
  runProperties[PROP_FIND_MISSING] = true;
if (switches.find("-e") != none || switches.find("--exclude") != none)
  runProperties[PROP_EXCLUDE] = true;
if (switches.find("-d") != none || switches.find("--dryrun") != none)
  runProperties[PROP_DRYRUN] = true;
// ...
```
