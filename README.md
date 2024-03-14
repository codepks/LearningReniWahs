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
	//using fetchStudentFactory = std::add_pointer<CartMan::IStudent*()>::type;
        typedef CartMan::IStudent* (*fetchStudentFactory)();
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

**Explanation**

```hubModule = LoadLibrary(L"StudentInterface.dll");```
This line declares a variable named hubModule to hold a handle to a module (a library or executable file). <br>
It's initially set to nullptr to indicate that no module is loaded yet.

```using fetchStudentFactory = std::add_pointer<CartMan::IStudent*()>::type;```
The above statement is same as : ```using fetchStudentFactory = std::add_pointer<CartMan::ITeacher*()>::type;``` and ```typedef CartMan::ITeacher* (*fetchStudentFactory)();```

```auto pfnMyFunction = (fetchStudentFactory)GetProcAddress(hubModule, "fetchObject");```
This block attempts to locate a specific function named "initPlugin" within the loaded library.<br>
The function pointer retrieved from GetProcAddress is typically a generic pointer type (e.g., void*). This doesn't provide any information about the function's arguments and return value, which can lead to errors if used incorrectly.

<br> One canuse the below instead
```
void* pfnMyFunction = GetProcAddress(hubModule, "fetchTeacher");
CartMan::ITeacher* (*actualFetchTeacher)(void) = (CartMan::ITeacher*(*)())pfnMyFunction;
```
```GetProcAddress()```: A Windows API function that retrieves the address of a named function from the specified module.<br> it it returns `void*` pointer

```FARPROC``` : A pointer type for holding function addresses.


# Handling Exception 

We can use macro to catch exceptions this ways:

```
// ExceptionFile.cpp

#pragma once
#define	TRY_API		try							\
		 	{								
#define	CATCH_API   	} 							\
			catch (std::exception& e)				\
			{  							\
				printf("Exception '%s' caught in : %s", e.what(), __FUNCTION__);	\
			}										\
			catch (...)									\
			{										\
				printf("Unknown exception caught in : %s", __FUNCTION__);		\
			}
```

You can use file in general:
```
#include <iostream>
#include <vector>
#include "ExceptionFile.h"

void generateVectorError() 
TRY_API
{
	std::vector<int> vec{ 1,2,3,4 };

	for (int i = 0; i < 5; i++)
	{
		std::cout << vec.at(i);
	}
}
CATCH_API

int main() {

	generateVectorError();
	return 0;
}
```

# Decoding using a macro    

```
#include <iostream>
#define __GetNextFlowBYTE__(ptr) *ptr; if ((unsigned char)*ptr < 0x80 || (unsigned char)*ptr >= 0xc0) return '?'; else ++ptr;

class CString {
public:
    unsigned DecodeUTF8(const char*& src);
};

unsigned CString::DecodeUTF8(const char*& src)
{

    unsigned char char2 = __GetNextFlowBYTE__(src);
    // Additional processing or handling of the decoded character can be performed here
    return char2;
}

int main()
{
    const char* utf8String = "\xE2\x82\xAC"; // UTF-8 encoded Euro symbol (€)
    CString myString;

    unsigned decodedChar = myString.DecodeUTF8(utf8String);
    std::cout << "Decoded character: " << decodedChar << std::endl;

    return 0;
}
```

**Explaination**
#define __GetNextFlowBYTE__(ptr) *ptr; if ((unsigned char)*ptr < 0x80 || (unsigned char)*ptr >= 0xc0) return '?'; else ++ptr; <br>

In this line, the macro __GetNextFlowBYTE__ is invoked with the src pointer as an argument. The macro expands to the corresponding code and performs the following actions: <br>


The value at the src pointer is dereferenced and assigned to the char2 variable.
If the value of decodedChar is less than 0x80 or greater than or equal to 0xc0, implying that it is not a valid UTF-8 continuation byte, the function will return a question mark '?'.

Making change in ptr also change decodedChar as both are pointing to same address

# Running a project executable
This is a different type of running :
If a project compiles to a .exe file and 
- If you want to run that project like a script
- And you want to make the script run while building some other solution

Then here are some ways to explore that:
1. Explicitly mention the PostBuildEvent in the .vcxproj file . Here BunJsonToSql is a generated .exe file
```
  <Target Name="BunJsonToSql" AfterTargets="PostBuildEvent">
    <Exec Command="bsonTosql.exe -i &quot;$(SolutionDir)\Assets\jsonTosql\infinam\500_devices.json&quot; -j &quot;$(SolutionDir)\CtrlCell\Source\Alerts.json&quot; -c &quot;$(SolutionDir)\Assets\API\shared\Static_InfiniAM.sql&quot; -l &quot;$(SolutionDir)\Assets\jsonTosql\sql\Static_InfiniAM_Last.sql&quot;" WorkingDirectory="$(OutDir)" />
  </Target>
```

2. Make a C# file generate a .bat script in the project Properties -> Build Events -> Pre/Post-Build/Link Events : E.g. ```call "C:\Users\pk152268\source\repos\Development2\CtrlCell\Source\alertmap_generator.bat"```

# Loading one .dll from another .dll file

Suppose if you want to run a A.dll from B.dll file
Way 1 : 
1. Create a function `SetupLogger()` in `A` solution to dynamically load B.dll library using LoadLibrary("A.dll")
2. Trigger that function call in dllmain.cpp under ```DLL_PROCESS_ATTACH:```

 Way 2 :
 1. Create a header file in `A` solution which can a function called `SetupLogger()`
 2. Create another file in B workspace and `#include<>` that file, insde that file create a struct/class -> which calls that `SetupLogger()` function
 3. Now make a static variable out of that structure which gurantees the loading of the dll being taken care as long as the program exists

```
//LoggerIni.cpp
#include "shared/LoggerSetup.h"

//=========================================================================================================================!

struct BoggerSetup
{
	BoggerSetup()
	{
		setupBoggerForRenAMP();
	}
};
static BoggerSetup firstThing; 
```

> **How above code works ?**
> A static variable declared in a file which is initialized in data segment of the code before the start of the main function.
> Also it should be decalred in .cpp file only
