# Cesium for Unity (v1.23.1) – Developer Setup on Ubuntu 22.04 with Unity 2022.3

This guide documents a Linux-specific developer setup for **Cesium for Unity v1.23.1** on **Ubuntu 22.04** with **Unity 2022.3 LTS**. The upstream Cesium developer setup is the starting point, but on Linux a few extra adjustments are needed so that:

- **Reinterop** is published in a form Unity accepts.
- the Cesium assembly is compiled on Linux.
- the generated `DotNet/...` bridge headers appear correctly.
- the native C++ plugin can be built for either **Editor** or **Standalone**.

---

## System Environment

- **OS:** Ubuntu 22.04.5 LTS
  - Architecture: 64-bit
  - Windowing System: X11
  - GNOME Version: 42.9
- **Unity Editor:** 2022.3.62f3
- **Unity Hub:** 3.11.1
- **.NET SDK installed:** `.NET 10.0.106`
- **Cesium for Unity:** `v1.23.1`

---

## Original Documentation

The official starting point is Cesium for Unity's `developer-setup.md`:

- https://github.com/CesiumGS/cesium-unity/blob/main/Documentation~/developer-setup.md

This README overrides a few steps for Linux compatibility and for a working native build flow on Ubuntu.

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

## 5. Open the Project in Unity to Generate Reinterop Output

Open the Unity project from Unity Hub.

Expected behavior:

- Unity compiles the Cesium C# side.
- Reinterop runs during compilation.
- native bridge code is generated under `native~/generated-*`.

You may see `DllNotFoundException` for the native plugin at this stage. That is expected before the native C++ plugin is built.

After Unity finishes compiling, close it and check for generated folders:

```bash
cd /path/to/YourUnityProject/Packages/com.cesium.unity
find native~ -iname "*dotnet*" -type d
find native~ -maxdepth 2 -type d -name "generated-*"
```

### Expected result after only opening Unity

Usually you will first get:

```text
native~/generated-Editor/include/DotNet
native~/generated-Editor/src/DotNet
```

That means the **Editor** interop bridge has been generated.

### If no `DotNet` folders appear

Try forcing Reinterop to rerun by touching the configuration files:

```bash
cd /path/to/YourUnityProject/Packages/com.cesium.unity
echo "// force reinterop" >> Runtime/ConfigureReinterop.cs
echo "// force reinterop" >> Editor/ConfigureReinterop.cs
```

Then reopen Unity and let it recompile.

### Important: how to get `generated-Standalone`

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
find native~ -maxdepth 2 -type d -name "generated-*"
```

You should now have a **`generated-Standalone`** folder as well.

You can discard the temporary built game afterward. The important result is that Unity has generated the **Standalone** Reinterop bridge.

### Why generating `generated-Standalone` is recommended

For this Linux setup, **starting with the Standalone/runtime path is recommended** because the Standalone native build is the one needed for the final delivered game, and in practice it is the more generally useful native target to prepare first. If you generate `generated-Standalone` early, your runtime native build can proceed cleanly and you avoid later confusion about why `-DEDITOR=OFF` fails.

---

## 6. Fix `UnityWebRequestAssetAccessor.h` and `.cpp`

On this Linux toolchain, the Cesium native source uses `std::optional` and `std::make_optional` in `UnityWebRequestAssetAccessor.h` and `UnityWebRequestAssetAccessor.cpp`, but those files do not explicitly include `<optional>`.

That can compile on some environments if another header indirectly pulls `<optional>` in by accident, but it fails here because the files are not self-contained enough for this compiler/include order.

### Why this fix is required precisely

Without this fix, the native C++ build fails with errors like:

- `std::optional in namespace std does not name a template type`
- `std::make_optional is not a member of std`
- `_maybeResponse does not exist`

Those are cascade errors caused by the missing `<optional>` include.

So before building native code, patch both files:

```bash
cd /path/to/YourUnityProject/Packages/com.cesium.unity

sed -i '19i #include <optional>' native~/src/Runtime/UnityWebRequestAssetAccessor.h
sed -i '29i #include <optional>' native~/src/Runtime/UnityWebRequestAssetAccessor.cpp
```

Optional verification:

```bash
sed -n '14,22p' native~/src/Runtime/UnityWebRequestAssetAccessor.h
sed -n '24,32p' native~/src/Runtime/UnityWebRequestAssetAccessor.cpp
```

---

## 7. Create a Custom Linux vcpkg Triplet

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

## 8. Build the Native Plugin

This section is the key Linux-specific part.

The CMake flag `-DEDITOR=...` determines **which generated Reinterop bridge** and **which native target type** CMake will build against.

### What `-DEDITOR` means precisely

- `-DEDITOR=ON`
  - Build the **Editor native plugin**
  - Uses **`generated-Editor`**
  - Intended for Editor-specific native integration

- `-DEDITOR=OFF`
  - Build the **Standalone/runtime native plugin**
  - Uses **`generated-Standalone`**
  - Intended for the actual runtime/player build
  - This is the recommended main target to prepare first on Linux

So if you only have `generated-Editor`, a build with `-DEDITOR=OFF` will fail because the Standalone-generated `DotNet/...` headers do not exist yet.

---

### 8A. Build type 1: `build-Standalone` (recommended)

Use this after you have generated `generated-Standalone` from Unity by doing a temporary Unity player build once.

```bash
cd /path/to/YourUnityProject/Packages/com.cesium.unity/native~

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

This is the build path that matters for the final delivered game. In this Linux workflow it is the recommended target to establish early, because it is the runtime-side native library you ultimately need for shipping the game.

Once Unity has generated `generated-Standalone`, this is the preferred build to use.

---

### 8B. Build type 2: `build-Editor`

Use this if you specifically want to build against the Editor-generated bridge.

```bash
cd /path/to/YourUnityProject/Packages/com.cesium.unity/native~

cmake -B build-Editor -S . \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DVCPKG_TRIPLET=x64-linux-unity \
  -DVCPKG_OVERLAY_TRIPLETS=$(pwd)/vcpkg/triplets \
  -DEDITOR=ON \
  -DCMAKE_POLICY_VERSION_MINIMUM=3.5

cmake --build build-Editor --target install --parallel $(nproc)
```

### What this build is for

This is the **Editor** native plugin build. It uses the `generated-Editor` bridge.

Use it when you want the explicit Editor target, or when you are testing purely editor-side native integration.

---

## 9. Restart Unity

After the native build installs successfully, reopen the Unity project.

At that point Unity should be able to load the installed native plugin instead of throwing `DllNotFoundException`.

---

## 10. Done

With these changes:

- Reinterop is published in a form Unity can load.
- Cesium is allowed to compile on Linux.
- the `DotNet/...` bridge headers are generated correctly.
- `UnityWebRequestAssetAccessor.*` is patched for this Linux toolchain.
- you can build either:
  - an **Editor** native plugin with `-DEDITOR=ON`
  - a **Standalone/runtime** native plugin with `-DEDITOR=OFF`

For this Ubuntu/Linux workflow, the recommended path is:

1. prepare Reinterop
2. open Unity once
3. do one temporary Unity player build to force `generated-Standalone`
4. patch `UnityWebRequestAssetAccessor.*`
5. build native code with **`build-Standalone`**

That gives you the most useful runtime-native setup first, while still leaving the Editor build path available when needed.