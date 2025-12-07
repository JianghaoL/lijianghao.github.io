# Unity Built-in Audio System Design

Hello! I'm Jianghao, lead of the audio team of the game *Sisyphus's Worst Day*. Over many projects that I have worked on, I've seen either beautifully or poorly designed audio systems.

Audio is one of, if not the most, important aspect of a game. Therefore, I'm writing this article to discuss a clean Unity built-in audio system set-up.

>All of the code presented in this article can be found at the link below, and is licensed under ***MIT License***, where you are free to use, modify, even distribute the code:
https://github.com/JianghaoL/Unity_BuiltIn_Audio_System

Now, let's dive into the journey of building our audio system!

## What is Unity's Built-in Audio System, Really?

Before discussing anything else, we need to understand the surface-level logic behind Unity's built-in audio workflow.

We will ignore the **Audio Mixer** for now (that's a mxing topic for another day). Unity's native audio system can be broken down into **2** core parts:<br>

  - **Audio Clip** - Essentially just **data**. Imported audio assets stored under your *Assets* folder.
  - **Audio Source** - A **component** you attach to a GameObject. The thing that actually *plays* audio.<br>

To play a sound, you drag an Audio Clip to an Audio Source's **Audio Clip** field. At taht point, you can think of an Audio Source as a "bus" on a mixing console - it can apply effects and finally output the clip.<br>

![AudioSource](Audio%20System%20Pic/Unity%20Audio%20Source.png)<br>
For a smaller scale Unity project, this simple setup works perfecty:<br>

*Attach an AudioSource → Reference it in a script → Call*
```csharp
AudioSource.Play()
```

But as your project grows, a single sound-emitting object might eventually need to play *different* Audio Clips at different moments. This means:
- Your logic scripts start to hold references to many Audio Clips
- A single GameObject may need multiple Audio Sources
- References are found everywhere
- The entired porject becomes **tightly coupled** to Audio Clips and Audio Sources.

All of this is *terrible* for project maintainability.

So, let's build a system that's cleaner, more unified, and nicely ***de***coupled.

## Time to Refactor!

The core problem here is:
>Every Audio Source must store references to all Audio Clips it might ever play.

So we ask:
>What if we could **centralize all Audio Clip references,** and manage playback through a dedicated **Audio Manager singleton**?

Then the gameplay scripts would no longer need to store Audio Clips directly. They'd simply pass a string (that refereces the name of an audio clip) and an Audio Source to the Audio Manager.

With this idea in mind, let's build it.

<br>

### Step 1 - Storing Audio Clip Data

We start by creating a **ScriptableObject** called `AudioConfigSO`.<br>

It contains:
- `allClips` - a **public** list of all audio entries
- `_dictionary` - a private lookup table

the list `allClips`, in fact, is a list of type `AudioConfig`, which is defined as:
```csharp
[Serializable]
public class AudioConfig
{
    public string key;
    public AudioClip clip;
}
```
![AudioConfigSO](Audio%20System%20Pic/AudioConfigSO.png)

`GetClip()` finds an Audio Clip by its custom name, which you set up in Unity Editor.

![GetClip()](Audio%20System%20Pic/GetClip%20Method.png)

In Unity, the ScriptableObject looks like this:

![AudioConfigsSOinUnity](Audio%20System%20Pic/AudioConfig%20Inspector.png)

<br>

### Step 2 - The Audio Manager

We then build the `AudioManager`, which is implemented as a **singleton**.

![AudioManager](Audio%20System%20Pic/AudioManager.png)

It holds a reference to `AudioConfigSO`, allowing it to fetch clip data.

We can add methods like:
- `Play`
- `Stop`
- `SetVolume`, etc.

![AudioManagerMethods](Audio%20System%20Pic/Play%20and%20Stop%20Methods.png)

>Note that each method receives an Audio Source as a **parameter** 

This is essential since the Audio Manager itself does ***not*** own any Audio Source components. Gameplay scripts still pass in whichever Audio Source they want to use.

At this point, we've achieved the core architecture:

- Query AudioClips by string
- Manage playback through a centralized AudioManager
- Gameplay scripts stay clean and decoupled.

Example usage becomes pleasant to look at:

![FunctionalScriptWithString](Audio%20System%20Pic/Functional%20Script%20with%20string.png)

## But, Can We Take it Even Further?

**Absolutely.**

As you can see, righ now we're still manually typing strings.<br>
Hard-coded strings in a large project?<br>
Probably not something you'd want.

>As a matter of fact, **never** put hard-coded strings in your project.

So, we can extend the Unity Editor.

Our goal now becomes:

**Automatically generate classes containing `const string` keys** for every entry in every `AudioConfigSO`.

That we, we get:
- Auto-complete (Yay!)
- Compile-time safety
- No more typos
- No more magic strings

Let's write a class that will do this:

```csharp
public static class GenerateAudioID
{
    [MenuItem("Tools/GenerateAudioID")]
    public static void Generate(){...}

    private static void CreateClass(AudioConfigSO audioConfigSO){...}
}
```

The `Generate()` method will scan:
`Resourcs/Audio Configs`

![GenerateMethod](Audio%20System%20Pic/Generate%20Method.png)

It loops through all assets, and for each `AudioConfigSO`, it calls `CreateClass()`, which creates a new class containing all of its keys.

![CreateClass](Audio%20System%20Pic/CreateClass%20method.png)

Now, you can do<br>
**Tools → GenerateAudioID**<br>
to generate class(es) for each of your `AudioConfigSO`.

![CreatedAudioID](Audio%20System%20Pic/AudioID%20example.png)

With this, we can further update our gameplay scripts by simply writing:

```csharp
audioManager.Play(Level_1_Audio_ID.Door_Close, audioSource);
```

Or as shown in the picture below:

![FunctionalScriptWithClassRef](Audio%20System%20Pic/Functional%20Script%20with%20Class%20ref.png)

Super clean, isn't it?

>Note that the generated class inherits from `AudioID`

Because of this inheritance, you can also use **reflection** to gather them automatically - this can be useful if, say, you want to load `AudioConfigSO` files based on level number, for example.

## Wrapping Up

This custom audio system:
- Solves the chaos of Unity's default Audio Source workflow
- Eliminates the scattering Audio Clip references
- Cleans up gameplay logic scripts
- Supports automatic code generation
- Prevents hard-coded strings.

Again, you can find all code licensed under **MIT License** here:
https://github.com/JianghaoL/Unity_BuiltIn_Audio_System

Maybe I'll dive into FMOD / Wwise system architecture as well.

Stay tuned!

