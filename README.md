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
