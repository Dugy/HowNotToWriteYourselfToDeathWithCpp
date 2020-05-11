# How not to Write Yourself to Death with C++
C++ is a very practical programming language. It has plenty of features to prevent all kinds of errors on syntactic level, is so fast that it rarely needs to be optimised... but it's said that too much code is needed to accomplish trivial tasks.

That's usually a consequence of overly low-level library interfaces, often designed with C compatibility and maximum performance in mind. However, a clean interface that needs minimal amount of code for typical usage is often more important. Designing such interfaces tends to be more difficult because they often can't be clean if they follow the low level behaviour. It requires much more skill than implementing the actual functionality.

Proper use of various features other languages were designed not to have allows creating classes that can be used with even less code than the same in high-level interpreted languages.

This is a series of lessons about various less known features of C++ that can be used to greatly reduce the amount of code needed to solve a task. The lessons are in markdown, designed to be converted to slides using [Marp](https://github.com/marp-team/marp).

## Coverage
Stuff that will be explained:
* How to achieve the same functionality with less code
* How to take your time to accelerate a lot of future development
  * Less code in the future
  * Less future errors to deal with
* A lot of C++ tricks to achieve unusual functionality

Stuff that is **not** the goal:
* Advanced algorithms
* Optimising code for speed
* Code style

## Requirements
* Experience with programming
* Knowledge of C++ sufficient to write common stuff without checking solutions online
* Understanding of object oriented design

Being able to read C++ code and having written a few Python scripts with intense googling is **not enough.**

## Outline
1. [Introduction](https://github.com/Dugy/HowNotToWriteYourselfToDeathWithCpp/blob/master/slides/01_Introduction.md)
2. [Object model in C++](https://github.com/Dugy/HowNotToWriteYourselfToDeathWithCpp/blob/master/slides/02_Object_model_cpp.md)
3. [Nontrivial operator usage](https://github.com/Dugy/HowNotToWriteYourselfToDeathWithCpp/blob/master/slides/03_Nontrivial_operator_use.md)
4. [Generic functions and SFINAE](https://github.com/Dugy/HowNotToWriteYourselfToDeathWithCpp/blob/master/slides/04_Generic_functions_and_SFINAE.md)
5. [Clean interface](https://github.com/Dugy/HowNotToWriteYourselfToDeathWithCpp/blob/master/slides/05_Clean_interface.md) (WiP)
6. Generic classes (TBA)
7. [Memory manipulation](https://github.com/Dugy/HowNotToWriteYourselfToDeathWithCpp/blob/master/slides/07_Memory_manipulation.md)
8. Runtime metaprogramming (TBA)
9. Template metaprogramming (TBA)
10. Putting it all together (TBA)
