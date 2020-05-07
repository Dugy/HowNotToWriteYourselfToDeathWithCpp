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
Implement the `parseCsv()` function, but add also flags for throwing with any mistake in file (otherwise it tries to go through), for throwing when the file is missing (otherwise it returns an vector) and a unified one for throwing in case of any of the earlier two mistakes (and do this one sparingly and without writing the offsets of the earlier flags twice).

```C++
auto data = parseCsv("experimentalData.csv", ParseFlags::PEDANTIC | ParseFlags::width(4));
```

---
## Design
At this point, you should know a load of tricks how to write less code. Now it's time to sit down and reflect about their usage.
