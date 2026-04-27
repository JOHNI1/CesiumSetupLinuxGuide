# Cesium for Unity (v1.23.1) – Developer Setup on Ubuntu 24.04 with Unity 6.3 LTS

This guide documents a Linux-specific developer setup for **Cesium for Unity v1.23.1** on **Ubuntu 24.04** with **Unity 6.3 LTS**.

The upstream Cesium developer setup is the starting point, but on Linux a few extra adjustments are needed so that:

- **Reinterop** is published in a form Unity accepts.
- the Cesium assembly is compiled on Linux.
- the generated `DotNet/...` bridge headers appear correctly.
- the native C++ plugin can be built correctly for **Editor** first, and then for **Standalone/runtime**.

---

## Critical clarification before you start

There are **two different native plugin targets** in this setup, and they are **not interchangeable**:

- **Editor native plugin**
  - built with `-DEDITOR=ON`
  - uses `generated-Editor`
  - required for the **Unity Editor process itself**
  - required to stop Editor-time Cesium/Reinterop initialization errors such as:
    - `DllNotFoundException: CesiumForUnityNative`
    - `The native library is out of sync with the managed one`

- **Standalone/runtime native plugin**
  - built with `-DEDITOR=OFF`
  - uses `generated-Standalone`
  - required for the **final built Linux game/player**
  - **not sufficient** to make the Unity Editor work correctly

### Important correction to a common misunderstanding

For Linux development, you should **build the Editor native plugin first**.

Do **not** start by prioritizing the Standalone build if your immediate goal is simply to open the Unity project and work with Cesium inside the Unity Editor.

Why:

- Unity Editor startup loads the **Editor** native plugin, not the Standalone one.
- If you only build `build-Standalone`, Unity Editor may still fail because `Packages/com.cesium.unity/Editor/libCesiumForUnityNative.so` is missing, stale, or built from the wrong generated bridge.
- A Standalone build is for the **delivered Linux player**, not for the editor process.

So the correct sequence is:

1. prepare Reinterop
2. allow Cesium to compile on Linux
3. patch native CMake
4. open Unity once so it generates **`generated-Editor`**
5. build **Editor** native with `-DEDITOR=ON`
6. confirm Unity Editor works
7. only then prepare **Standalone/runtime** by forcing Unity to generate **`generated-Standalone`** and building with `-DEDITOR=OFF`

---

## System Environment

- **OS:** Ubuntu 24.04 LTS
  - Architecture: 64-bit
- **Unity Editor:** 6.3 LTS
- **Cesium for Unity:** `v1.23.1`

---

## Prerequisite Packages on Ubuntu

If you get a Reinterop error about the `dotnet` SDK being missing or the wrong version, install the .NET SDK explicitly.

If native CMake compilation fails because standard compiler/build tools are missing, install `cmake` and `build-essential`.

`nasm` is also recommended because it can be used for faster JPEG decoding in native dependencies.

```bash
sudo apt update
sudo apt install git cmake nasm build-essential dotnet-sdk-10.0
```

---

## Original Documentation

The official starting point is Cesium for Unity's `developer-setup.md`:

- https://github.com/CesiumGS/cesium-unity/blob/main/Documentation~/developer-setup.md

This README overrides a few steps for Linux compatibility and for a working native build flow on Ubuntu.

---

## 0. Add cesium to package-lock.json

inside "dependencies":  
add the package `"com.cesium.unity": "file:com.cesium.unity",`

---

## 1. Clone the Repository into your Unity Project

We typically place Cesium inside the Unity project's `Packages/` folder.

Example:

```bash
cd /path/to/YourUnityProject/Packages
git clone --recurse-submodules -b v1.23.1 git@github.com:CesiumGS/cesium-unity.git com.cesium.unity
```

This clones the repository into a folder named `com.cesium.unity`, including all submodules.

---

## 2. Adjust `Reinterop.csproj` in `Reinterop~`

Unity on Linux may reject the default Reinterop project configuration if it targets a framework or language mode that Unity's Roslyn setup does not accept. Replace `Reinterop~/Reinterop.csproj` with the following:

```bash
cd /path/to/YourUnityProject/Packages/com.cesium.unity
gedit Reinterop~/Reinterop.csproj
```

Replace the file contents with:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <LangVersion>latest</LangVersion>
    <IsRoslynComponent>true</IsRoslynComponent>
    <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
    <CompilerGeneratedFilesOutputPath>Generated</CompilerGeneratedFilesOutputPath>
  </PropertyGroup>

  <ItemGroup>
    <Compile Remove="RoslynIncrementalGenerator.cs" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="3.8.0" PrivateAssets="all" />
  </ItemGroup>

  <!-- Package the built DLL as a Roslyn analyzer/source generator -->
  <ItemGroup>
    <None Include="$(OutputPath)\$(AssemblyName).dll" Pack="true" PackagePath="analyzers/dotnet/cs" Visible="false" />
  </ItemGroup>

</Project>
```

---

## 3. Allow Cesium to Compile on Linux in `CesiumForUnity.asmdef`

Open:

```bash
gedit /path/to/YourUnityProject/Packages/com.cesium.unity/Source/CesiumForUnity.asmdef
```

Make sure of the following:

1. add `"LinuxStandalone64"` to `includePlatforms`
2. make sure `defineConstraints` does **not** exclude Linux editor builds

Recommended relevant section:

```json
{
    "name": "CesiumForUnity",
    "rootNamespace": "",
    "references": [
        "Unity.InputSystem",
        "Unity.Mathematics",
        "Unity.Splines"
    ],
    "includePlatforms": [
        "Android",
        "Editor",
        "iOS",
        "macOSStandalone",
        "WSA",
        "WindowsStandalone64",
        "WebGL",
        "LinuxStandalone64"
    ],
    "excludePlatforms": [],
    "allowUnsafeCode": true,
    "overrideReferences": false,
    "precompiledReferences": [],
    "autoReferenced": true,
    "defineConstraints": [],
    "versionDefines": [
        {
            "name": "com.unity.splines",
            "expression": "1.0.0",
            "define": "SUPPORTS_SPLINES"
        }
    ],
    "noEngineReferences": false
}
```

This step is required because otherwise Unity may exclude Cesium from compilation on Linux, which prevents Reinterop from running and prevents the `DotNet` bridge headers from being generated.

---

## 4. Publish Reinterop

From inside `com.cesium.unity`:

```bash
cd /path/to/YourUnityProject/Packages/com.cesium.unity

dotnet publish Reinterop~ -o .
git restore Reinterop.dll.meta
```

This places `Reinterop.dll` in the package root so Unity can load it as a Roslyn analyzer/source generator.

---

## 5. Fix `native~/CMakeLists.txt` so `REINTEROP_GENERATED_DIRECTORY` follows `EDITOR`

After publishing `Reinterop.dll`, patch the native CMake configuration so it selects the correct generated bridge folder depending on whether you are building the **Editor** or **Standalone/runtime** native plugin.

Open:

```bash
gedit /path/to/YourUnityProject/Packages/com.cesium.unity/native~/CMakeLists.txt
```

Find this line:

```cmake
set(REINTEROP_GENERATED_DIRECTORY "generated-Editor" CACHE STRING "The subdirectory of each native library in which the Reinterop-generated code is found.")
```

Replace it with:

```cmake
if (EDITOR)
  set(REINTEROP_GENERATED_DIRECTORY "generated-Editor")
else()
  set(REINTEROP_GENERATED_DIRECTORY "generated-Standalone")
endif()
```

### Why this step is required

By default, the native CMake file is hardcoded to use `generated-Editor`. That is acceptable only when building with `-DEDITOR=ON`.

But for a **Standalone/runtime** build with `-DEDITOR=OFF`, the native build must consume the headers and source files from **`generated-Standalone`**, not `generated-Editor`.

Without this patch, CMake keeps looking in the editor-generated bridge directory even when you are trying to build the runtime plugin. That causes mismatches and leads to missing `DotNet/...` headers or to a build that silently points at the wrong generated bridge.

This patch makes the generated bridge directory follow the meaning of the `EDITOR` flag correctly:

- `EDITOR=ON` → use `generated-Editor`
- `EDITOR=OFF` → use `generated-Standalone`

---

## 6. Create a Custom Linux vcpkg Triplet

Cesium ships triplets for several targets, but not the Linux Unity target we need here. Create a custom one:

```bash
cd /path/to/YourUnityProject/Packages/com.cesium.unity
gedit native~/vcpkg/triplets/x64-linux-unity.cmake
```

Paste:

```cmake
include("${CMAKE_CURRENT_LIST_DIR}/shared/common.cmake")

# Custom triplet for x64 Linux with Unity-specific settings.
set(VCPKG_TARGET_ARCHITECTURE x64)
set(VCPKG_CRT_LINKAGE static)
set(VCPKG_LIBRARY_LINKAGE static)
set(VCPKG_CMAKE_SYSTEM_NAME Linux)
set(VCPKG_LIBRARY_PREFIX "")
```

---

# Part A — Build the Editor native plugin first

This section is the **required first native build** for Linux development in Unity.

If your immediate goal is to:

- open the Unity project successfully
- stop editor-side Reinterop/Cesium initialization failures
- use Cesium inside the Unity Editor

then this is the native target you must build first.

---

## 7. Open the Project in Unity to Generate Editor Reinterop Output

Open the Unity project from Unity Hub.

Expected behavior:

- Unity compiles the Cesium C# side.
- Reinterop runs during compilation.
- native bridge code is generated under `native~/generated-*`.

You may still see `DllNotFoundException` for the native plugin at this stage. That is expected **before** the native C++ plugin is built.

After Unity finishes compiling, close it and check for generated folders:

```bash
cd /path/to/YourUnityProject/Packages/com.cesium.unity
find native~ -iname "*dotnet*" -type d
find native~ -maxdepth 2 -type d -name "generated-*"
```

### Expected result at this stage

After only opening Unity, you usually want to see:

```text
native~/generated-Editor/include/DotNet
native~/generated-Editor/src/DotNet
```

That means the **Editor** interop bridge has been generated.

### If no `DotNet` folders appear

Try forcing Reinterop to rerun by touching the configuration files:

```bash
cd /path/to/YourUnityProject/Packages/com.cesium.unity
echo "// force reinterop" >> Source/Runtime/ConfigureReinterop.cs
echo "// force reinterop" >> Source/Editor/ConfigureReinteropEditor.cs
```

Then reopen Unity and let it recompile.

---

## 8. Build the Editor native plugin

Now build the native plugin for the **Unity Editor process**.

```bash
cd /path/to/YourUnityProject/Packages/com.cesium.unity/native~

rm -rf build-Editor
cmake -B build-Editor -S . \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DVCPKG_TRIPLET=x64-linux-unity \
  -DVCPKG_OVERLAY_TRIPLETS=$(pwd)/vcpkg/triplets \
  -DEDITOR=ON \
  -DCMAKE_POLICY_VERSION_MINIMUM=3.5

cmake --build build-Editor --target install --parallel $(nproc)
```

### What this build is for

This is the **Editor native plugin** build.

It must pair with:

- `generated-Editor`
- `-DEDITOR=ON`

This is the native library Unity Editor itself needs in order to initialize Cesium/Reinterop correctly on Linux.

### Why this build must come first

If you build only `build-Standalone` and skip `build-Editor`, Unity Editor can still fail because the Editor plugin remains missing, stale, or mismatched.

Typical editor-side failure symptoms include:

- `DllNotFoundException: CesiumForUnityNative`
- `The native library is out of sync with the managed one`

---

## 9. Restart Unity and verify the Editor now works

After the Editor native build installs successfully, reopen the Unity project.

At this point, Unity should be able to load the installed Editor native plugin instead of throwing editor-time Cesium/Reinterop initialization errors.

Do not move on to Standalone/runtime until this works.

---

# Part B — Standalone/runtime build for the delivered Linux game

This section is **separate** from the Editor setup.

Use this part only **after** the Unity Editor is already working correctly with the Editor native plugin.

This is the build path for the **final Linux player/runtime package** of your game that uses Cesium.

---

## 10. Generate `generated-Standalone` by doing one temporary Unity player build

Opening the Unity project is usually enough to create **`generated-Editor`**, but **not** necessarily **`generated-Standalone`**.

To get `generated-Standalone`, you must temporarily make a Unity player build once:

1. open the project in Unity
2. go to **File → Build Settings**
3. select **Linux** as the target platform
4. click **Build**
5. choose any temporary output folder
6. wait until Unity has finished the player build preparation

After that, re-check:

```bash
cd /path/to/YourUnityProject/Packages/com.cesium.unity
find native~ -maxdepth 2 -type d -name "generated-*"
```

You should now have a **`generated-Standalone`** folder as well.

You can discard the temporary built game afterward. The important result is that Unity has generated the **Standalone** Reinterop bridge.

### Why this step is required

A Standalone/runtime native build with `-DEDITOR=OFF` must consume **`generated-Standalone`**.

If that folder does not exist yet, the runtime native build will fail or will point at the wrong bridge.

---

## 11. Build the Standalone/runtime native plugin

After `generated-Standalone` exists, build the runtime native plugin:

```bash
cd /path/to/YourUnityProject/Packages/com.cesium.unity/native~

rm -rf build-Standalone
cmake -B build-Standalone -S . \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DVCPKG_TRIPLET=x64-linux-unity \
  -DVCPKG_OVERLAY_TRIPLETS=$(pwd)/vcpkg/triplets \
  -DEDITOR=OFF \
  -DCMAKE_POLICY_VERSION_MINIMUM=3.5

cmake --build build-Standalone --target install --parallel $(nproc)
```

### What this build is for

This is the **Standalone/runtime** native plugin build.

It must pair with:

- `generated-Standalone`
- `-DEDITOR=OFF`

This native plugin is for the **final Linux game/player runtime**, not for the Unity Editor process.

---

## 12. Final result

With these changes:

- Reinterop is published in a form Unity can load.
- Cesium is allowed to compile on Linux.
- the generated `DotNet/...` bridge headers are generated correctly.
- the **Editor** native plugin is built first and used by the Unity Editor.
- the **Standalone/runtime** native plugin is built later and used by the final Linux player.

---

## Recommended exact workflow summary

For this Ubuntu/Linux workflow, the recommended path is:

### Stage 1 — Make Unity Editor work

1. clone Cesium into `Packages/com.cesium.unity`
2. patch `Reinterop~/Reinterop.csproj`
3. patch `Source/CesiumForUnity.asmdef`
4. publish `Reinterop.dll`
5. patch `native~/CMakeLists.txt`
6. create `native~/vcpkg/triplets/x64-linux-unity.cmake`
7. open Unity once to generate **`generated-Editor`**
8. build native with **`build-Editor`** and **`-DEDITOR=ON`**
9. reopen Unity and confirm the editor works

### Stage 2 — Prepare the delivered Linux player/runtime

10. do one temporary Unity Linux player build to force **`generated-Standalone`**
11. build native with **`build-Standalone`** and **`-DEDITOR=OFF`**
12. use that runtime plugin path for the final delivered Linux game

---

## Troubleshooting note about the most common misunderstanding

If Unity Editor is throwing errors but you already built `build-Standalone`, that does **not** prove the Editor native setup is correct.

A successful Standalone build only proves that the **runtime/player** native plugin path was built.

It does **not** replace the need for the **Editor** native plugin.

If the Editor still fails, go back and verify:

- `generated-Editor` exists
- you built with `-DEDITOR=ON`
- the installed Editor plugin was updated
- you did not accidentally leave a stale `Packages/com.cesium.unity/Editor/libCesiumForUnityNative.so` in place from an older build