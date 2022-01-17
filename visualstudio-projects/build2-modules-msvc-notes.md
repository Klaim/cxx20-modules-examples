

# 2022-01-16

I found some time to work on the msvc module support.

First I upated my Visual Studio project files building the `build2` modules example project.
Just updating it to VS2022, to the 143 runtime and adding the hello-simple project that you added recently made it work as before, which kind of prove that at least all these cases works with latest msvc (stable). There is only one project I didn't setup which is the utility one, see below.

I had to do some changes because when I initially setup this, I changed all the module interface files (`.mxx`) and the module partition files to `.ixx` for convenience, it helped not having to add the flag specifying which one it is. For the purpose of `build2` however we need to use these flags anyway so I reverted the renamings and added the flags manually (which lead to some ambiguity issues, see below).

## Build Logs

I set MSBuild logging verbosity to "detailed" for build log files (default is "minimal", there is also "diagnostic" but I suppose that would be too much noise).
I put the resulting build logs in this google drive (you can download all it's content as 1 zip using the download button - tell me if you have issues with this):
    https://drive.google.com/drive/folders/1A7WX44kClmAp7hoI9XnmC_DP5KdtfyPS?usp=sharing

These logs contains the command lines using `cl.exe` and a lot more, like the properties being set/considered and the whole logic of deduction of what to build and in which order (or at least that looks like it to me). Some of the operations are opaque, at least in appearance, probably because they are internal to MSBuild.
Tell me if that's not enough to deduce some information. Maybe you want the .json that are generated too?

## Overview

- All the projects are set with `/std:c++20`, not with `/std:c++lastest` to avoid interference from WIP features.
- Flags identifying C++ source files (replacing `/TC` and `/TP` in the case of module files):
    - Module interface files (with `export module mymodule;` ) must compile with `/interface`. I did not find it in the doc (it still refers to `/experimental:module` etc.).
    - Module partions (both private or interface - with `export module mymodule:mypartition;` or `module mymodule:mypartition`) must compile with `/internalPartition` (see the `hello-partition` case). Not in the doc either yet.
    - Module interface partitions (with `export module mymodule:mypartition;`) can also compile with `/interface` (see `hello-partition` case).

- In VS projects, to be able to use standard library headers as import and to enable some other automatic modules detection, we need to set the option (on projects or c++ translation units):

    > "Scan Sources For Module Dependencies" which is explained as (with my notes) "Makes the build scan all C++ source files [from the project?], not just module interfaceand header unit sources, for module and header unit dependencies and build the full dependencies graph".

    This is "no" by default, I had to set it to "Yes" on all projects. This is not a compiler flag and so far I didn't spot what it does exactly, but this is at the build-system level so I suppose that's the same work `build2` does when looking for dependencies?

- To force all `#include` to be handled as `import` , use the flag `/translateInclude`. I didn't find more "precise" flags that would state for example that only standard headers should be translated. See: https://docs.microsoft.com/en-us/cpp/build/reference/translateinclude?view=msvc-170

- I did not use explicitly the `/headerUnit` flag but you might want to take a look: https://docs.microsoft.com/en-us/cpp/build/reference/headerunit?view=msvc-170

- I did not use explicitly the `/reference[[...]]` flag either but it should be used to specify the ifc files to use for the compilation of a C++ file: https://docs.microsoft.com/en-us/cpp/build/reference/module-reference?view=msvc-170

- In the VS interface there are new fields for clarifying which modules (directories) are or not visible from other targets depending on the current one (in msvc "target" should be translated to "project"). Here is how it looks for me: https://ibb.co/12HDgN7  Not sure this is relevant though.

- Related to assembly files, maybe this is interesting (not sure its specific to c++) for the linker: https://docs.microsoft.com/en-us/cpp/build/reference/assemblymodule-add-a-msil-module-to-the-assembly?view=msvc-170

- Not used explicitly and not in the doc, but there is `/ifcSearchDir ...` which is described as search paths to resolve import directives, see: https://ibb.co/7kcbKxy

- The `/exportHeader` flag appears in header translations compilation, and we can use that instead of `/TC`, `/TP`, `/interface` or `/internalPartition` if I understand correctly, but it was not clear for me when it's necessary to set manually so I didn't. It's for example used automatically when translating standard headers to modules (see below).


From the build log I deduced the following:

- Here is how `string_view` is compiled when compiling `hello-simple`:
    ```
    C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.30.30705\bin\HostX64\x64\CL.exe /c /I"C:\Program Files (x86)\Visual Leak Detector\include" /I"E:\tools\vcpkg\installed\x64-windows\include" /ZI /JMC /nologo /W3 /WX- /diagnostics:column /sdl /FS /Od /D _DEBUG /D _CONSOLE /D _UNICODE /D UNICODE /Gm- /EHsc /RTC1 /MDd /GS /fp:precise /Zc:wchar_t /Zc:forScope /Zc:inline /std:c++20 /permissive- /ifcOutput "x64\Debug\string_view_PUA4X9BBZ1GRUBHI.ifc" /Fo"x64\Debug\string_view_PUA4X9BBZ1GRUBHI.obj" /Fd"x64\Debug\vc143.pdb" /sourceDependencies "x64\Debug\string_view_PUA4X9BBZ1GRUBHI.ifc.d.json" /external:W3 /Gd /exportHeader /FC /errorReport:prompt  /TP "C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.30.30705\include\string_view"
    ```
    We can see that `/ifcOutput` is used to specify the module interface file.

- To compile `hello.mxx` :
    ```
    C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.30.30705\bin\HostX64\x64\CL.exe /c /I"C:\Program Files (x86)\Visual Leak Detector\include" /I"E:\tools\vcpkg\installed\x64-windows\include" /ZI /JMC /nologo /W3 /WX- /diagnostics:column /sdl /FS /Od /D _DEBUG /D _CONSOLE /D _UNICODE /D UNICODE /Gm- /EHsc /RTC1 /MDd /GS /fp:precise /Zc:wchar_t /Zc:forScope /Zc:inline /std:c++20 /permissive- /ifcOutput "E:\Projects\build2-libs\msvc-build2-modules\cxx20-modules-examples\visualstudio-projects\hello-simple\x64\Debug\hello.mxx.ifc" /sourceDependencies:directives "x64\Debug\\" /Fo"x64\Debug\hello.mxx.obj" /Fd"x64\Debug\vc143.pdb" /external:W3 /Gd /interface /FC /errorReport:prompt  /TP "..\..\hello-simple\hello.mxx"
    ```
    - We can see `/sourceDependencies:directives`, which from the doc is supposed to be the flag generating the jsons containing the module dependencies for this file: https://docs.microsoft.com/en-us/cpp/build/reference/sourcedependencies-directives?view=msvc-170
    - we can see the `/interface` flag as expected and the `/ifcOutput`.

- To compile `hello.cxx`:
    ```
    C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.30.30705\bin\HostX64\x64\CL.exe /c /I"C:\Program Files (x86)\Visual Leak Detector\include" /I"E:\tools\vcpkg\installed\x64-windows\include" /ZI /JMC /nologo /W3 /WX- /diagnostics:column /sdl /FS /Od /D _DEBUG /D _CONSOLE /D _UNICODE /D UNICODE /Gm- /EHsc /RTC1 /MDd /GS /fp:precise /Zc:wchar_t /Zc:forScope /Zc:inline /std:c++20 /permissive- /sourceDependencies:directives "x64\Debug\\" /Fo"x64\Debug\\" /Fd"x64\Debug\vc143.pdb" /external:W3 /Gd /TP /FC /errorReport:prompt "..\..\hello-simple\hello.cxx"
    Tracking command:
        C:\Program Files\Microsoft Visual Studio\2022\Community\MSBuild\Current\Bin\amd64\Tracker.exe /d "C:\Program Files (x86)\MSBuild\15.0\FileTracker\FileTracker32.dll" /i E:\Projects\build2-libs\msvc-build2-modules\cxx20-modules-examples\visualstudio-projects\hello-simple\x64\Debug\hello-simple_MD.tlog /r E:\PROJECTS\BUILD2-LIBS\MSVC-BUILD2-MODULES\CXX20-MODULES-EXAMPLES\HELLO-SIMPLE\HELLO.CXX /b MSBuildConsole_CancelEventff5494a31f9443bea386bda6327790e8  /c "C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.30.30705\bin\HostX64\x64\CL.exe"  /c /I"C:\Program Files (x86)\Visual Leak Detector\include" /I"E:\tools\vcpkg\installed\x64-windows\include" /ZI /JMC /nologo /W3 /WX- /diagnostics:column /sdl /FS /Od /D _DEBUG /D _CONSOLE /D _UNICODE /D UNICODE /Gm- /EHsc /RTC1 /MDd /GS /fp:precise /Zc:wchar_t /Zc:forScope /Zc:inline /std:c++20 /permissive- /sourceDependencies:directives "x64\Debug\\" /Fo"x64\Debug\\" /Fd"x64\Debug\vc143.pdb" /external:W3 /Gd /TP /FC /errorReport:prompt "..\..\hello-simple\hello.cxx"
    ```
    I am not sure what is the tracking command but it seems related to dependency tracking. I also have the `.tlog` directories if you want to take a look.

I'll stop here for now, there is a lot in these files.

## hello-partition

There is a slight ambiguity (or I'm misunderstanding something) with this one.
Here we have 3 module files:

- `hello.mxx` is a module interface -> I set the `/interface` flag.
- `hello-format.mxx` is a module interface partition if you look in the code, but for `build2` it is a module interface -> I set the `/interface` flag.
- `hello-printer.mxx` is a private module partition if you look at the code, but for `build2` it is a module interface -> I set `/internalPartition` flag.


To me this is ambiguous: to me `hello-printer.mxx` is not an interface unit so it should not have the `.mxx` extension. MSBuild agrees with me, that is if I rename it with `.cpp` it will be compiled correctly with the auto-detection.
In any way, compiling `hello-format.mxx` with `/internalPartition` also works, but `hello-printer.mxx` will only compile with `/internalPartition`.
Not sure if there is a difference with either flag in the case of `hello-format.mxx`, but in doubt I decided to stay with `/interface` for any module interface partition.

I think I woudl prefer to use `.cpp` or `.cxx` for private module interfaces as they are not interfaces, so the choice of extensions in the `hello-partition` project looks weird to me.


## hello-header-import

It built but I realized there is a workaround gcc's issue with header import, so I commented that to make sure `import <string_view>;` is used.

## hello-library-header

Almost all projects build with and without `/translateInclude`, which implies compiling part of the standard library as module headers as expected.

The `libhello-header-shared` does not export symbols by default so I added the necessary plumbing for symbol export/import.


# hello-header-translate

This compiles with and without `/translateInclude` and also with and without "Scan Sources For Module Dependencies" set to "Yes".

# hello-utility-library-module

I did not make a Visual Studio project for this one. I suspect only the flags to build the utility library changes something.


