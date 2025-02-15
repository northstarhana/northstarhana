---
tags:
  - CPlusPlus
  - UnrealEngine
---
If you're reading this, you should know by now that you should [stop live coding](Stop%20Live%20Coding.md). 

Reinstancing causes rampant asset corruption and sets up false expectations for designers less familiar with the intricacies of the relationship C++ has with the Unreal Editor. 

Hot Reload is simply a less-reliable version of Live Coding + Reinstancing (if you can believe it) that exists as a remnant of the pre-Live++ UE4 days. It gets silently enabled when Live Coding is disabled - most often when users hear word that Live Coding (with Reinstancing on) is no good or if the developer is using Linux where Live Coding isn't supported and thus Hot Reload is forced on.

In this article we'll talk about solutions for stopping yourself and other team members from playing with fire and risking corrupting a project's Blueprints.

# Method 1: Turning Reinstancing off by Default

This is the simplest method of taming Unreal's live patching systems to make it less error-prone, but is also the most prone to interference itself.

In your project, create or open `Config/DefaultEditorPerProjectUserSettings.ini` and add or change the following lines:
```ini
[Script/LiveCoding.LiveCodingSettings]
bEnableReinstancing=False
```

Now by default Reinstancing will be disabled for all users of your project. This is the easiest method which doesn't require changing any code, but it comes with a few drawbacks:

* This only disables Reinstancing. Hot Reload will still be sneakily enabled if a user unticks bEnableLiveCoding in the Editor
* Nosy designers can easily just tick the box to turn it back on in their local Editor settings. Yay asset corruption! 🥳

# Method 2: Permanently Remove Hot Reload and disable Reinstancing Entirely

This is the most comprehensive method for preventing users from accessing the aforementioned dangerous hotpatch operations but requires that you are on a Source Build of the engine and know how to share this modified build with your team. 

> [!Important]
> You will still be able to use Live Coding with this method! Live Coding without Reinstancing can be used for the following:
> 
> * Editing function bodies
> * Adding new non-virtual non-reflected functions
> * Editing non-reflected function signatures
> * Adding static variables
> * Adding new non-reflected classes and structs
>   
>  Users will be able to turn Live Coding on/off through their Editor Preferences without the risk of enabling Hot Reload through a checkbox, but the Reinstancing option will be eliminated entirely.

> [!DANGER]
> If you are not on a Source Build (i.e. your Unreal Engine installation came from the Epic Games Launcher or from the Linux Binaries webpage), do not attempt this method. Without a full souce build, modifying Engine files is not supported and there is no guarantee that you will be able to compile these changes, leaving your Engine install in a broken state where you'll have to Verify and potentially redownload.

> [!WARNING]
> These modifications will trigger a full Engine rebuild of ~4700 actions. Don't do this if you need to use your Editor right now!

## Removing the `WITH_HOT_RELOAD` Preprocessor Define to Compile Without Hot Reload

In `Engine/Source/Runtime/Core/Public/Misc/Build.h`, find the following section:
```c++
/**
* Whether we support hot-reload. Currently requires a non-monolithic build and non-shipping configuration.
*/
#ifndef WITH_HOT_RELOAD
	#define WITH_HOT_RELOAD (!IS_MONOLITHIC && !UE_BUILD_SHIPPING && !UE_BUILD_TEST && !UE_GAME && !UE_SERVER)
#endif
```

Comment out or delete the existing line and replace with `#define WITH_HOT_RELOAD 0`
```c++
/**
* Whether we support hot-reload. Currently requires a non-monolithic build and non-shipping configuration.
*/
#ifndef WITH_HOT_RELOAD
	// #define WITH_HOT_RELOAD (!IS_MONOLITHIC && !UE_BUILD_SHIPPING && !UE_BUILD_TEST && !UE_GAME && !UE_SERVER)
#define WITH_HOT_RELOAD 0
#endif
```

This is not enough to disable compiling support for Hot Reload. UHT's codegen has a flag in `UhtCompilerDirective` to ignore preprocessor directives when parsing header files. We need to remove the exemption for `WithHotReload` in `Engine/Source/Programs/Shared/EpicGames.UHT/Parsers/UhtHeaderFileParser.cs`:
```c#
		/// <summary>
		/// The following flags are always ignored when keywords test for allowed conditional blocks
		/// </summary>
		AllowedCheckIgnoredFlags = CPPBlock | NotCPPBlock | ZeroBlock | OneBlock | WithHotReload,
```

Remove `WithHotReload` from the flag:
```c#
		/// <summary>
		/// The following flags are always ignored when keywords test for allowed conditional blocks
		/// </summary>
		AllowedCheckIgnoredFlags = CPPBlock | NotCPPBlock | ZeroBlock | OneBlock,
```

With that, Hot Reload support will be completely stripped out of all configurations of your Engine project. Good riddance!
## Permanently Disabling Reinstancing

In `Engine/Source/Developer/Windows/LiveCoding/LiveCodingSettings.h`, find the following UPROPERTYs
```c++
	UPROPERTY(config, EditAnywhere, Category=General, Meta=(EditCondition="bEnabled"))
	bool bEnableReinstancing;
	
	UPROPERTY(config, EditAnywhere, Category=General, Meta=(EditCondition="bEnabled", DisplayName="Automatically Compile Newly Added C++ Classes"))
	bool bAutomaticallyCompileNewClasses;
```

Comment them out or delete them. I recommend commenting them out and leaving a note as to why they were removed.
```c++
	// Reinstancing can cause asset corruption and unexpected behavior. Do not use this.
	// UPROPERTY(config, EditAnywhere, Category=General, Meta=(EditCondition="bEnabled"))
	// bool bEnableReinstancing;

	// Without Reinstancing we cannot compile new classes.
	// UPROPERTY(config, EditAnywhere, Category=General, Meta=(EditCondition="bEnabled", DisplayName="Automatically Compile Newly Added C++ Classes"))
	// bool bAutomaticallyCompileNewClasses;
```

`bAutomaticallyCompileNewClasses` is used in the `Tools > New C++ Class...` New Class Wizard. Without Reinstancing it's always going to fail, so we remove the option here. Later, we will add a dialogue box to warn users who are used to this workflow about these changes.

In `Engine/Source/Developer/Windows/LiveCoding/Private/LiveCodingModule.cpp`, find the following lines:
```c++
bool FLiveCodingModule::IsReinstancingEnabled() const
{
#if WITH_EDITOR
	return Settings->bEnableReinstancing;
#else
	return false;
#endif
}

bool FLiveCodingModule::AutomaticallyCompileNewClasses() const
{
	return Settings->bAutomaticallyCompileNewClasses;
}
```

Now they should always return `false`:
```c++
bool FLiveCodingModule::IsReinstancingEnabled() const
{
	return false;
}

bool FLiveCodingModule::AutomaticallyCompileNewClasses() const
{
	return false;
}
```

## Adding a Warning for Developers that use the New Class Wizard

Some developers prefer to use the New Class wizard to create class files as it includes file templates for common Unreal classes where vanilla Visual Studio provides none (Rider does provide templates in `Add > Unreal Engine Class...`) This still works with these changes however without Reinstancing the New Class wizard will never be able to hotpatch the classes that it adds.

To add a dialogue to warn them and offer to stop or continue, add the following in `Engine/Source/Editor/GameProjectGeneration/Private/GameProjectUtils.cpp` at the top of `GameProjectUtils::OpenAddToProjectDialog`:
```c++
void GameProjectUtils::OpenAddToProjectDialog(const FAddToProjectConfig& Config, EClassDomain InDomain)
{
	EAppReturnType::Type NoCompileMsgResult = FMessageDialog::Open(EAppMsgCategory::Error, EAppMsgType::YesNo, LOCTEXT("NewClassDialogCannotCompileForYou", "Reinstancing is disabled in this version of the Engine. New source files can be added to the filesystem from the New Class menu however new classes will not be compiled while the Editor is running.\n\nIf you still want to use the New Class menu, close the Editor once the classes are added to the filesystem and compile from your IDE.\n\nWould you like to continue?"));
	if (NoCompileMsgResult == EAppReturnType::No)
		return;
	
	//...
}
```

## The Result

Now we can compile (in our IDE 🥰), sit back, and watch the entire Engine rebuild.

To confirm that Hot Reload is completely gone, we can go into Editor Settings and untick `Enable Live Coding`. When we click the hotpatch button in the bottom right of the Editor we should see the following warning in the log:
```
Warning: RebindPackages not possible (hot reload not supported)
```

As for the New Classes menu, before the New Classes dialogue appears we'll be greeted with an informational dialogue box that explains the changes and offers to stop or continue.

Hooray! Now you can rest easy knowing that your fellow team members won't accidentally corrupt a class with a derived BP with over 100 carefully placed nodes and 20 meticulously-configured components with Hot Reload/Reinstancing.