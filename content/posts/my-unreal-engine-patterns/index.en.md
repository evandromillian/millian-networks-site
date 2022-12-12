---
title: "My Unreal Engine Patterns"
date: 2022-12-11T14:15:08-03:00
description: ""
draft: false
author: "Evandro Millian"
resources:
- name: "featured-image"
  src: "featured-image.webp"
- name: "featured-image-preview"
  src: "featured-image-preview.webp"
tags: ["Unreal Engine"]
categories: ["Game Development"]
lightgallery: true
---

The newest version of the Unreal Engine is growing to be the most powerful engine available to the public. It contains many improvements presented through the years in papers and conferences like the GDC.

But, due to be too big, it's important to know some good practices to be able to produce more in less time, or to have less problems or errors. These are the basic patterns I use for my Unreal Engine projects.

This list will be updated based on the learnings made.

### General Tips

* Always comment your code, be per snippet, in the declaration (C++) or event/function declaration (Blueprint)
* Events/functions should have a single responsibility

### Objects Structure

* Pay attention with strong references in blueprints
    * Each reference to a concrete Blueprint type is an increase of memory
    * Try to move the reference type as close as possible to Unreal built-in types, like **Character**, **Actor**, **ActorComponent**, etc
    * Use **Soft References** to classes and load them at runtime using one of the methods below:
        * **Async Load Class Asset** needs less configuration to use
        * **Async Load Primary Asset** needs more configuration, but its more flexible

### Blueprints

* Do not use multiple input exec 
    * What may happens if you need to override the **Initialize** event in a children blueprint? The **BeginPlay** event will execute the code its linked, not the **Initialize** event. This will lead to bugs in the children blueprint.

    {{<figure src="/images/unreal-engine-patterns/fork.webp" >}}

    * Avoid as much as possible to have multiple input exec. In real code (that will be probably much bigger), it will be hard to be sure about the code flow and the event/function result. Simplicity is key, always.
    
* Refactor a complex event into a simpler one
    * The **Sequence** node is great to organize the flow of your event.
    * Break big events in multiple minor events/functions
    * Repeated code should also be moved to its own function
* Don't be afraid to use **Set Timer By Event/Function Name** in functions
    * It's better for organization, as delegate will be in its own event/function
* Use multiple inbound to a exec with caution, avoid most of the times
    * This can lead to confusion, especially if one of the events is overridden
* Prefer **DoOnce** instead of boolean flags
    * If **DoOnce** is opened/closed in many places, create custom events calling **Open** and **Close** instead of use long **Exec** links.
* Prefer **Select** instead of **Branch** to select values
    * Beyond the code flow simplification, **Select** allows to handle multiple values and is configurable for many data types (even enums).

{{<figure src="/images/my-unreal-engine-patterns/selects.png" width=600 >}}

### C++

* Implement most of the behavior in C++, exposing extension points to Blueprint, as examples:
    * Basic **Character** functionality
    * **Actor Components**
    * Integrations with online services for account control
    * Expose to Blueprints some funcionally that it's only available in C++
    * Wrap long running tasks in **latent functions** (tied to Actors) and **async tasks** (instances of **BlueprintAsyncActionBase**, that are reusable)

* **DuplicateObject** only copies variables market with **UPROPERTY()**
* To create a function that has more than one return value, just add function arguments by reference

{{< highlight cpp >}}

UFUNCTION(BlueprintCallable)
float ReturnMultipleValues(FString& TextRet);

{{< /highlight >}}

### C++ to Blueprint Magic

* Use multiple inputs (**UFUNCTION** with meta **ExpandEnumAsExecs**) to reduce the need of **Switch** and **Branch** nodes in Blueprint code

{{< highlight cpp >}}
UENUM()
namespace EAuthorityType
{
	enum Type
	{
		Authority,
		Remote,
		Standalone
	};
}

...

UFUNCTION(BlueprintCallable, meta = (ExpandEnumAsExecs = "OutResult"))
void SwitchAuthorityType(AActor* Actor, TEnumAsByte<EAuthorityType::Type>& OutResult);
{{< /highlight >}}

{{<figure src="/images/my-unreal-engine-patterns/multiple_outputs.png" width=400 >}}

* Use **DetermineOutputType** with **DynamicOutputParam** to dynamically update BP function output type , without the need to write a custom K2Node

{{< highlight cpp >}}
UFUNCTION(BlueprintCallable, meta=(DeterminesOutputType="ActorClass"))
AActor* GetActorOfClass(const UObject* WorldContextObject, TSubclassOf<AActor> ActorClass);

UFUNCTION(BlueprintCallable, meta=(DeterminesOutputType="ActorClass", DynamicOutputParam="OutActors"))
void GetAllActorsOfClass(TSubclassOf<AActor> ActorClass, TArray<AActor*>& OutActors);
{{< /highlight >}}

### Widgets

* Implement optimizations from Unreal Engine documentation.
    * Use event-driven updates instead of bounded attributes
    * Break complex widgets into smaller ones
    * Use **Spacers** instead of **Size Boxes** when possible
    * Avoid combine **Scale Boxes** and **Size Boxes**
    * ~~Use Rich Text Widgets sparingly~~ Using **Common Text**
    * Pay attention to the animation costs according to the following table:

Cost                         | Examples
-----------------------------|--------------------------------------------------------------
No CPU Cost/Least Expensive  | Material-only animations.
Low CPU Cost                 | Blueprint-scripted animations that do not require Sequencer.
High CPU Cost                | Sequencer animations made with UMG's animation editor.
Most Expensive CPU Cost      | Animations that cause layout changes.

* About animations, we'll use **CommonActivatableWidgetStack** transitions most of the time.

### Automation and Validations

* To improve the quality of the assets, reduce errors and increase development speed, Unreal Engine provide some tools as:
    * [Asset Validators](https://www.youtube.com/watch?v=zRZjNN6jxCI)
    * [Asset/Actor Action Utilities](https://www.youtube.com/watch?v=wJqOn88cU7o&list=PLoObU30LCLpcItHySX3dpP02CDUZw4XNk&t=926s)
    * [Blutility Script Buttons](https://www.youtube.com/watch?v=wJqOn88cU7o&list=PLoObU30LCLpcItHySX3dpP02CDUZw4XNk&t=1360s)
    * [Editor Utility Widgets](https://www.youtube.com/watch?v=wJqOn88cU7o&list=PLoObU30LCLpcItHySX3dpP02CDUZw4XNk&t=1604s)