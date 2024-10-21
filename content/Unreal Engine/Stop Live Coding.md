---
title: "Stop Live Coding: The broken promise of a Unity-like workflow in Unreal"
tags:
  - Beginner
  - UnrealEngine
  - CPlusPlus
  - CSharp
---
If you're new to Unreal and watch any YouTube tutorial or worse, pay for an Unreal course you'd be forgiven for thinking that the sane workflow for adding new C++ classes and modifying them would look something like:

1. Right click "C++ Classes" in the content drawer
2. Click "New C++ Class"
3. Name our class and let the Editor "compile" while it's running!

This even looks similar to the Unity workflow with C# scripts, perfect for new users migrating from Unity! Until you run into your first Live Coding bug:

> Why doesn't my new member/property/field show up when I "compile"?

> Why don't my changes persist after restarting the Editor?

> Why is my Editor crashing?

> Why are my assets corrupted?

These are symptoms of using this broken workflow that is often recommended by Unreal educators that don't understand the practical limitations of Live Coding and hotpatching a compiled language like C++.
# Compiled Language? Managed Language? What's the difference?

C++ is a **statically compiled language** that is converted into machine code to be executed directly on the CPU. This means that the compiler must know the size and layout of all types ahead of time in order to generate the machine code around them. The size and layout of a type cannot change after the program has been compiled as there is no abstraction layer to allow adding/removing parts of memory safely. 

Languages like C#, Java, JavaScript, and even Blueprint/Kismet are **managed/intermediate/intepreted languages**. These languages only compile down to bytecode and read by a virtual machine/interpreter that translates these bytecode calls into chunks of machine code that the CPU understands. This can be done either at runtime (known as Just-In-Time compilation or JIT) or at compile time (known as Ahead-of-Time or AOT compilation). This more modular abstraction layer lends itself to features like being able to dynamically swap out code and modify types at runtime as we aren't dealing with memory nearly as directly as a compiled language like C++. The downside to this more abstract approach is the cost of translating these calls to machine code, resulting in poorer performance compared to a compiled language.

# Wait, if C++ is static and can't be changed at runtime, how does Live Coding work?

Out of the box when you press ctrl+alt+f11 to invoke Live Coding after changing a class, two notable things happen:

* New patched executables are created and linked into the running Editor. Existing functions have their first instruction replaced with a jump instruction to the new function.

> [!tip] Extra Info
> Live++, the technology Epic licenses for Live Coding, requires the `/hotpatch` flag (enabled by default by Unreal) which makes the first instruction in the function at least 2 bytes long. This ensures that Live++ can insert the aforementioned jump instruction. 

* Any UObjects with new or removed UPROPERTY/UFUNCTION members have a new temporary counterpart instanced and then reflected members are copied over to the new object. Functions that use this object type are similarly redirected to the new type.

> [!info]
> For further reading about how Reinstancing works, check out `FReloadClassHelper::ReinstanceClass`

Thus we have avoided the impossible task of modifying static code at runtime at the cost of creating a Rube Goldberg machine of temporary copies and ferrying memory between 2 different types. The limitations are, in order of severity, as follows:

* If we only change a constructor it will not be ran again. At best we can get the constructor to run on the new temporary object as a side effect if we arbitrarily modify something else. 
* Unreal doesn't know about non-reflected members so they will not be transferred over to the new object reliably.
* By adding/removing variable and virtual function members (due to the vtable used to resolve which function to call) we are changing the physical layout of the object in memory but any code that isn't aware of the change to the temporary copy type will just rush headlong into it expecting to find a member at a specific location and the Editor will crash.
* There is a chance that garbage or duplicate property information will be copied over to the new copy which if saved to the asset will **permanently** corrupt it.

The last point is the most common symptom of Live Coding and there is no way to tell if we have it until you see duplicate components on your Actor BPs, phantom properties/components that won't go away even after they are removed, and other unexpected behavior. For the most part there is no way to recover an asset from this.

> [!tip]
> Duplicate components can be potentially fixed with [this plugin](https://github.com/Duroxxigar/ComponentPointerFixer). Not all hope is lost!

# A Better Unreal C++ Workflow

Getting on-track to avoid ad-hoc project corruption starts with re-establishing our relationship with the Editor. **The Unreal Editor is a standalone application that we compile Modules for for so that designers and artists can build a video game**. It should be treated as any other C++ program in that it would be extremely abnormal to compile an application while it is running. **There is still a place for Live Coding to be used** but it should never be the primary way to introduce compiled code into our application. 

## Starting the Editor

Simply opening the .uproject file from our filesystem will prompt us to compile the project if the `Development Editor` build config is not up-to-date. This will silently build this config in the background and launch it when it is ready. There is nothing inherently wrong with this but it will become cumbersome when we need more powerful tools for development like [an attached debugger](Basic%20Debugging%20for%20Unreal.md). This option should most commonly be used by Blueprint developers, level designers, and artists in the absence of distributed Editor binary builds for our project as we simply have a better option. These instructions will be for Jetbrains Rider and Visual Studio as these are the only two fully-supported IDEs for Unreal development.

> [!Warning]
> Ensure that there are no Unreal Editor processes running otherwise you may encounter an UnrealBuildTool error.

### Rider
* Right click your .uproject file 
* `Open With > Jetbrains Rider`
* Once the project is loaded, press alt+F5 or the green "Bug" icon to build and run
	* This attaches a debugger as well. For more information see [this article](Basic%20Debugging%20for%20Unreal.md).
* If the build succeeds with no compile errors the Editor will open automatically

### Visual Studio
* Right click your .uproject file and click  `Generate Visual Studio Project Files`
* Once the project is loaded press F5 or the green "Play" arrow that says "Local Windows Debugger" to build and run
	* This attaches a debugger as well. For more information see [this article](Basic%20Debugging%20for%20Unreal.md).
* If the build succeeds without errors the Editor will open automatically

## Changing Editor Settings


> [!DANGER]
> Do not disable Live Coding.

With the bad rep Live Coding has, your first instinct may be to disable it. 

Disabling Live Coding silently enables Hot Reload which was Epic's previous attempt at integrating hotpatching to C++ that had all of the problems of Live Coding plus more. What we actually want to disable is **Reinstancing** which is what fakes modifying UObject layouts at runtime which can cause permanent asset corruption. Reinstancing can be found in Editor Preferences under Live Coding. This can be set as a project-wide default for all users current and future by adding these lines to `Config/DefaultEditorPerProjectUserSettings.ini`

```ini
[/Script/LiveCoding.LiveCodingSettings]
bEnableReinstancing=False
```

We will see later that Live Coding can still be used with its Reinstancing capabilities disabled for a narrow (albeit useful) usecase.
## Adding a Class

> [!Warning]
> Ensure that there are no Unreal Editor processes running (if you ran the Editor previously, close it)

### Rider
* Right click a directory in the module we are working in (likely your Game module, the one named after your project in the solution tree)
* Click `Add > Unreal Class`
* Pick a parent class/template and press OK
* Press alt+F5 to compile. If the build succeeds with no compile errors the Editor will open automatically

### Visual Studio

Adding files in vanilla Visual Studio is tricky. Because the .sln is fake and does not actually show our real project structure, attempting to add a new file from the solution tree will instead add it to `Intermediate/` which is not actually where our source code is discoverable.
* Open your file explorer/terminal to where we want to add a new file
* Create the file in your filesystem
* Right click your .uproject file and click  `Generate Visual Studio Project Files`
* Press F5 to compile. If the build succeeds with no compile errors the Editor will open automatically

> [!Tip]
> In both Rider and Visual Studio triggering a build will kill the attached Unreal Editor process, build, and relaunch saving us a few clicks. Be wary however that this will not save pending changes in the Editor so be sure to save your assets before closing/compiling.

## Modifying a Class

Modifying an existing class is similar to adding a class in that you (almost always) must close the Editor before compiling. Simply add/remove members, modify functions, close the Editor, and build as described above in Adding a Class.

# Wait, so you're telling me I have to restart the Editor EVERY TIME I make a change!?

**Yes**\*. It is the only 100% safe way of working with C++ code in Unreal Engine. As you become more experienced with writing code and know what to expect when you make changes, spamming compile will become less and less necessary and your restarts will become hours apart.

## \* Except when you change function bodies

Remember earlier where we learned about how Live Coding works? Most of the snags of using Live Coding come from Reinstancing (which should be disabled now as prescribed by this guide). The one part of Live Coding that works reliably is **making changes to existing function bodies, excluding constructors**. 

That being said, if you already have a function `AMyActor::BeginPlay`, adding a few lines to `BeginPlay` and live coding with the Editor open will work reliably and at worse can only cause a crash with no permanent corruption **assmuming you have Reinstancing disabled**. 

This is particularly useful for gameplay code that *must* be spammed in order to see changes in the Play-In-Editor (PIE) window such as movement code, math code, or Slate widget code (or... ImGui 👉👈)

# Addendum

> [!warning]
> Opinions ahead!

Expecting gameplay programmers to write bespoke code in C++ is frankly ridiculous. Gameplay and other high-level systems are much better served by a scripting language such as C# or Lua that can be properly hot reloaded as seen in other game engines. Blueprint attempts to fill this niche but falls short in its lack of support for basic features and unwieldy API when it comes to making custom nodes on top of being binary asset based making source control a pain in the ass. Trying to program core systems in Blueprint is silly but C++ being the only full-featured text language to write gameplay code in is even sillier. Thankfully Tim has hinted that UE6 will integrate Verse, Epic's new scripting language used in UEFN. Until then, there is the community project [UnrealSharp](https://github.com/UnrealSharp/UnrealSharp) looking to bring C# with hot reload to Unreal as a self-contained plugin as well as [AngelScript](https://angelscript.hazelight.se/) which requires an Engine fork.