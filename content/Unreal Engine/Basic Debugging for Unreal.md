---
tags:
  - Beginner
  - UnrealEngine
  - CPlusPlus
---
Bugs happen to the best of us but the real knowledge check is how we go about diagnosing them.

There aren't many things in programming quite as ubiquitous and annoying as runtime bugs. Whether it's a math function that's always returning 0 no matter the input or a nullptr that runs into a `check` or `ensure` that pops up the Crash Reporter, you're going to face *thousands* over your tenure as an Unreal Engine programmer and *how* we go about fixing it can be just as important as discovering the fix itself. In this article, we are going to learn how to use the interactive debugger to diagnose these kinds of issues in a fraction of the time it would take to print debug or reach out to others for a code review.

>[!info]
> For the sake of example we will be using JetBrains Rider but Visual Studio has equivalent debugging capabilities and the principles taught apply to both.

# The Case of the "Access Violation"

Here we have a simple example of an Actor subclass that lets us configure a skeletal mesh that it will apply to itself when it hits the BeginPlay part of its lifecycle. Not terribly useful, but bear with me.

> [!warning]
> This code is intentionally written in an unsafe manner for the sake of example. Please do not use these samples in your project.


```c++
UCLASS()
class MYPROJECT54_API AMyMeshActor : public AActor
{
	GENERATED_BODY()

public:
	// Constructor (ctor)
	AMyMeshActor();

protected:
	// Components should never be Edit__, this does not affect our ability to edit
	// the component's properties. Just the pointer to the component. Edit__ can
	// cause serialization errors.
	UPROPERTY(VisibleAnywhere)
	TObjectPtr<USkeletalMeshComponent> MeshComponent;

	UPROPERTY(EditAnywhere)
	TObjectPtr<USkeletalMesh> MeshToSet;

protected:
	//~ AActor Interface
	virtual void BeginPlay() override;
	//~ End AActor Interface
}
```

```c++
// Constructor (ctor)
void AMyMeshActor::UMyClass()
{
}

// Function causing us to crash when this Actor begins play
void AMyMeshActor::BeginPlay()
{
	Super::BeginPlay();

	if (IsValid(MeshToSet)) 
	{
		MeshComponent->SetMesh(MeshToSet);
	}
}
```

We create a Blueprint subclass of this actor, configure `MeshToSet` to a valid mesh, drag the Blueprint into the world hit PIE, and... 

```
Exception 0xc0000000 encountered at address 0x000000000000: Access violation reading location 0x0000000000
```

Not what we wanted to see. Our first instinct might be to add some `UE_LOGFMT` statements to figure out what's going on...

```c++
void AMyMeshActor::BeginPlay()
{
	Super::BeginPlay();

	// The thing at the end is called a ternary statement. It is essentially a
	// shorthand if-else statement that says Condition ? IfTrue : Else
	UE_LOGFMT(LogTemp, Warning, "MeshComponent is {0}", IsValid(MeshComponent) ? "Valid" : "Invalid");
	UE_LOGFMT(LogTemp, Warning, "MeshToSet is {0}", IsValid(MeshToSet) ? "Valid" : "Invalid");

	if (IsValid(MeshToSet)) 
	{
		MeshComponent->SetMesh(MeshToSet);
	}
}
```

We [compile our code](Stop%20Live%20Coding), relaunch the Editor, start Play-In-Editor, and before we can even see our logs, the Editor crashes and the Crash Reporter is contacted.

We can't even keep the Editor running long enough to see our logs! Not only that but we have to compile just to hope that we'll see logs that give us more information. This method is called "print debugging" and it has its place as we will circle back to later, but at this point in order to get anything useful done we would need to start adding `IsValid` checks *around our temporary log tracing code* which is where this method starts to get ridiculous.

# A Better Way

> [!warning]
> The following assumes that you are familiar with the [safe C++ workflow](Stop%20Live%20Coding). This guide is not written with the unsafe Live Coding-based workflows often taught online in mind.

Thankfully, developers outside of the gamedev sphere have encountered the same roadblocks for decades and have provided us with a better way: **interactive debugging**. Using debug symbols which map execution points in our compiled code to lines of code that we can see in our IDE, we can precisely pause execution of our program at a specified line of code by placing **breakpoints**, allowing us to peek into the memory and just *see* the values for ourselves "frozen in time" so to speak. We can also use breakpoints to see if a chunk of code is even being ran by placing one inside a function and observing if execution halts or not.

## Installing Debug Symbols

In order to debug a program we need `.pdb` or "Program Database" files that map human readable lines of code in our source code to execution points in our binaries. In typical software development as well as compiling the Engine from source we would compile these ourselves using specific build configurations. This step covers Unreal Engine binary downloads from the Epic Games Launcher (often just called "EGL build" or "Binary build")

In the Epic Games Launcher Library tab where you can see your Engine versions, click the dropdown arrow next to **Launch** and then click **Options**. This will bring up the installation options for your Engine version. Next, check **Editor symbols for debugging** and click **Apply**.

> [!question] Why are the Engine symbols such a huge download?
> The Engine symbols typically range from 30-70GB depending on Engine version. This is simply because there is a *lot* of machine code in a compiled Unreal Engine build to map to *a lot* of source code. It is recommended to buy a separate 1+ TB SSD just for Unreal Engine development, as building the Engine from source (which you may very well need to do in the future) is a several-hundred-gigabyte affair.

## Switching our Project's Build Configuration

It should be no surprise that we also need reliable debug symbols for *our* code. By default, your solution will be set to `Development Editor` build config which you can see near the Build buttons at the edge of your IDE. This is the most optimized, fastest build configuration for our Editor and is useful if distributing Editor builds to designers and artists who may not have a full developer suite on their PCs. The problem is that the optimizations performed during compilation will cause our breakpoints to be missed and our memory inspection to show flat-out false data. With an EGL Engine build the best configuration for debugging available to us will be `DebugGame`. Switch the build config to `DebugGame Editor`. 

> [!note]
> By using a non-optimized debugging configuration we gain the ability to debug but lose performance. How much performance depends on your system, but I personally leave my build config on `DebugGame Editor` permanently to retain debugging capabilities at all times.

## Launching with the Debugger

Now we can simply build and launch the Editor with an attached Debugger. In Rider press `alt+F5` or click the green bug icon. In Visual Studio press `F5` or click "Local Windows Debugger". In Rider the Debugging window will pop up automatically. You are now interactively debugging.

## Catching Exceptions and Asserts with the Debugger

One of the biggest advantages of running with a debugger is that **the program no longer closes and displays the crash reporter when we hit an exception or assert**. If we start Play-In-Editor now the program will instantly freeze just like when it crashed... but it takes us to the line that caused the exception, we get a full callstack, and we can freely explore the memory at the time of the crash.

If we hover over `MeshComponent` we'll find our problem in seconds. It's nullptr and calling `->SetMesh(MeshToSet)` is dereferencing a nullptr.

> [!tip]
> `Obj->Foo()` is shorthand equivalent to `*Obj.Foo()`.

Why is our mesh component nullptr? Well, let's see where we initialized it in the constructor...

```c++
void AMyMeshActor::UMyClass()
{
}
```

We never initialized the variable with a live `USkeletalMeshComponent`. Let's go ahead and fix the code (and improve it while we're at it)

```c++
// Constructor (ctor)
void AMyMeshActor::UMyClass()
{
	MeshComponent = CreateDefaultSubobject<USkeletalMeshComponent>("MeshComponent");
	SetRootComponent(MeshComponent);
}

void AMyMeshActor::BeginPlay()
{
	Super::BeginPlay();

	if (IsValid(MeshToSet)) 
	{
		check(IsValid(MeshComponent)); // Should never be invalid at this point
		MeshComponent->SetMesh(MeshToSet);
	}
}
```

We close the Editor (the easiest way to do this is to press `shift+F5`/the red stop icon or simply press `alt+f5` to kill the process and rebuild + relaunch), compile, relaunch with the debugger, and start PIE. Assuming we set a mesh in `MeshToSet`, everything should work. But what if we want to be extra sure?

## Setting Breakpoints

If we want to look inside an object at runtime to confirm what its state it, we can set a **breakpoint**. As mentioned previously a breakpoint can be used to essentially cause a controlled exception at any time, freezing the program in time and allowing us to look inside our objects. To set a breakpoint in both Rider and Visual Studio, simply move the mouse to the left of your code (inside `BeginPlay` in our example) and click where a faint red circle appears. Now if execution hits this point, our program will halt execution and we will jump to this line of code in our IDE, allowing us to hover over variables and view the callstack. 

In this case, hovering over `MeshComponent` should now show a valid pointer to a `USkeletalMeshComponent` which we can click the dropdown to expand and look at its variables inside.

### The Editor is frozen, now what?

If you want to continue execution, assuming you set the breakpoint and caused the exception yourself, you can either click the green triangle resume button or press `F5`. To remove the breakpoint, simply click the red dot that you placed again. The program will resume as if nothing happened.

> [!danger]
> With a debugger attached you will be able to parse between access violations (nullptr exceptions), breakpoints, and asserts when they are hit. By hitting `F5` you are "stepping over" breaks. While it is usually safe to step over breakpoints, stepping over access violations and asserts can lead to unexpected behavior, especially if the break took you to Editor code. This can lead to corrupted assets. A good general rule of thumb is that if you don't understand what it does or why the break happened, **don't step over it**.


## On the Callstack

While we did not have to traverse the callstack in order to solve our example bug in this article, it is important to understand what it is. The callstack is essentially a sequential history of functions that were called to get to where we hit a break. By hitting the break we are taken to the top of the callstack. You can click any point of the callstack to go to where the state of the program was at that function call and inspect memory there. 

> [!tip]
> This is particularly useful for quickly understanding underdocumented code. Place a breakpoint in a function that you surmise is high up on the callstack and then follow it backwards to figure out how that function got called and the circumstances that surrounded it.

# A Note On Print Debugging

Despite the power interactive debugging gives you, there are still times where print debugging is necessary. For example, if you are debugging movement code and you want to see the velocity of a Character while it moves it would be much more valuable to use print debugging (or even better, the Gameplay Debugger or ImGui) to see the values change over time while you control the Character in PIE. Placing a breakpoint on a hot area of code will have you spamming `F5` to step over while being unable to control the Character in the Editor viewport.