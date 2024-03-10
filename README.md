# Create a DLL

We will try to create a an interface and expose the functions of it to create a 3d party dll.
Steps to do so:
1. Create an interface with pure virtual functions
2. Export that interface using __declspec(dllexport)
3. One can use macro to use __declspec(dllexport)
4. Create the implementation part of the interface
5. Export the object of the implementation using old C style method 
6. Make the requried changes in the CMAKE.text or visual studio properties to define the macro for export/import
7. Load the dll to use the API


## .dll file
This file is one solution made in Visual Studio
### Header File

- Using namespace makes sure that it is accessible to this file only
- Other files importing this will have their own copy of the contents of this class
- The functions/classes below are non- and cannot be accessed by other classes.
- Adding withing namespace scope makes a function local to a translation unit
- In "Solution -> Properties -> General -> Configuration Properties" one can set the output to be a dll
- This MODEL_ENGINE_EXPORTS is defined in Solution -> Properties -> C/C++ -> Preprocessor

**Exaplaining Macros**
- In the Visual Studio solution where I have defined the STUDENT_ENGINE_API macro in the solution-> proeprties, there STUDENT_ENGINE_API will be taken as dllexport else in the solutions where this macro is not defined int the properties it will take it as dllimport

```
#pragma once
#include <iostream>

#ifdef MODEL_ENGINE_EXPORTS
#	define STUDENT_ENGINE_API __declspec(dllexport)
#else
#	define STUDENT_ENGINE_API __declspec(dllimport)
#endif

namespace CartMan{
	class STUDENT_ENGINE_API IStudent	{
	public:
		virtual void setName(std::string name) = 0;
		virtual void setRoll(int roll) = 0;
		virtual std::string getName() = 0;
		virtual int getRollNum() = 0;
	};
};

//this is the exposure part
extern "C"
{
	extern STUDENT_ENGINE_API CartMan::IStudent* fetchObject();
}
```

## Hidden Implementation

```
#include <iostream>
#include <string>
#include "Header.h"

namespace CartMan
{
	class Student : public IStudent
	{
	public:
		static Student* getStudentObject()	{
			if (object == nullptr)
				object = new Student(0, "");
			return object;
		}

		~Student() {};

		void setName(std::string name) { this->name = name; };
		void setRoll(int roll) { this->rollNum = roll; };
		std::string getName() { return name; };
		int getRollNum() { return rollNum; };


	private:
		Student(int roll, std::string nameStu) {
			rollNum = roll; name = nameStu;
		};

		static Student* object;

		int rollNum;
		std::string name;

	};

	Student* Student::object = nullptr;
}

// Making a function to access the interface
CartMan::IStudent* fetchObject(){
	return CartMan::Student::getStudentObject();
}
```

## .exe file

Another solution of Visual Studio where we dynamically load the dll and use the Student Interface
```
#include <iostream>
#include <windows.h>
#include <filesystem>
#include <tlhelp32.h>
#include <filesystem>
#include "..\ConsumerProducer\Header.h"

HINSTANCE hubModule = nullptr;

CartMan::IStudent* getStudentObject()
{
    hubModule = LoadLibrary(L"StudentInterface.dll");

    if (hubModule != nullptr)
    {
        using fetchStudentFactory = std::add_pointer<CartMan::IStudent*()>::type;
        auto pfnMyFunction = (fetchStudentFactory)GetProcAddress(hubModule, "fetchObject");
        if(pfnMyFunction != nullptr)
        {
            CartMan::IStudent* fetchObject = pfnMyFunction();
            return fetchObject;
        }
    }
}


int main()
{

    CartMan::IStudent* object = getStudentObject();

    object->setName("Prashant");
    object->setRoll(77771111);
    std::cout << object->getName();
    std::cout << object->getRollNum();
}
```
