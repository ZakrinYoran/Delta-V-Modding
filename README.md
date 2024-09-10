##### About

This document will cover the basic process of getting started with mod development in ΔV: Rings of Saturn. This guide is written for Windows, but the applications used should also be available for macOS and Linux.

# Setup

This Section describes the basic process of setting up Godot to begin modding.

##### Downloads

 The following applications are the only ones required to get started.

- First, obviously [ΔV: Rings of Saturn](https://store.steampowered.com/app/846030/V_Rings_of_Saturn/), as we'll be using the decompiled game for reference.

- Second, the [Godot Game Engine](https://godotengine.org/download/3.x/), ΔV is built on Godot Version 3.5.3 as of writing this.

- Third, [Godot reverse engineering tools](https://github.com/bruvzg/gdsdecomp) (GDRE), which we will use to decompile ΔV.

Extract/move these somewhere useful, preferably not your downloads. 

##### Decompiling

The process for decompiling ΔV is fairly straightforward when using GDRE.

- Open the folder containing `gdre_tools.exe` in your file explorer.

- Open the ΔV game folder (on steam it's Right-Click -> Manage -> Browse Local Files).

- Copy `Delta-V.pck` (not `.exe`!) into the GDRE folder.

- Open GDRE and select "Recover Project" from the "RE Tools" menu.

- Select `Delta-V.pck` and wait for the file verification process to finish.

- Leave the default settings, set the folder location, and extract the project.

- Once the extraction process is finished, ΔV should be ready for import into Godot.

I have occasionally had issues with decompiling ΔV using GDRE's gui, so I've taken to running it through command line. If the above steps fail, try the following process:

- Copy `Delta-V.pck` into the GDRE folder as before.

- Open windows command prompt and `cd` into the folder (e.g. `cd C:\godotTools\GDRE_tools`)

- Run `gdre_tools.exe` with the following command: `gdre_tools --headless --recover=Delta-V.pck`. This will create a new folder called containing the decompiled game files.

- Move the decompiled files to wherever you want your project located.

For more detail on the usage of GDRE, see the [usage section of their readme](https://github.com/bruvzg/gdsdecomp?tab=readme-ov-file#usage).

##### Importing ΔV into Godot

Loading the decompiled game files into Godot is rather simple. 

- Open your copy of Godot and select the 'Import' option.

- Navigate to where you stored your decompiled version of ΔV and select the `project.godot` file.

- Launch the Delta-V Project, and let Godot import all of the files.

The initial import process can take a significant amount of time, depending on hardware. If it appears to freeze, don't worry, it is still processing. After the first import, future loading times will be significantly better.

# Creating A Simple Mod

This section will cover some of the basics for mod creation, I also recommend reading the [modding documentation](https://gitlab.com/Delta-V-Modding/Mods/-/blob/main/MODDING.md), though it is limited.

##### Basic Structure

The basic format of a mod is quite simple, create a new folder in the root of the project  (e.g. `res://ExampleMod/`) and add a script named `ModMain.gd`. This is the script that will be run by the ModLoader, and is our hook into the game. I recommend structuring your mod to mirror the folder structure of the vanilla game, both to indicate what portion of the vanilla game is affected, and to make overriding assets easier.

##### ModMain.gd

`ModMain.gd` is where initial mod setup takes place, loading assets, overriding vanilla logic, modifying translations, etc. To assist with this, the ModLoader contains some simple functions for scene/script manipulation and adding translations. Unfortunately, I found these functions to either be too limited, or in the case of `addTranslationsFromCSV()`, no longer functional. To remedy this, I have provided some replacements that I use on my own mods:

- `updateTL()` is used to load translations from a file using the [comma-separated values](https://en.wikipedia.org/wiki/Comma-separated_values) format.
  
  - `path:String` is the path to the translations file, relative to the root of the mod folder (e.g. `"i18n/translation.txt"`).
  
  - `delim:String` is an optional parameter indicating what string is used to separate the values of your translation file (defaults to `","` if no delimiter is provided).

- `installScriptExtension()` is used to override vanilla script behavior.
  
  - `path:String` is the path to the new script, relative to the root of the mod folder.
  
  - The [inherited](https://docs.godotengine.org/en/3.5/tutorials/scripting/gdscript/gdscript_basics.html#inheritance) script gets overridden, whenever it is used, your modified script will instead be applied.
  
  - Inheritance is very powerful, and can be used to override most vanilla behavior, with some exceptions.
  
  - Scripts must be extended before they are loaded. Once applied to a node, the script is locked in and modifying the script resource will not affect existing instances.

- `replaceScene()` overrides or replaces a vanilla scene.
  
  - `newPath:String` is the path to your modded scene, relative to the mod folder.
  
  - `oldPath:String` is an optional path to the vanilla scene you are overriding, if no path is provided, the same path as your new scene will be used, but relative to `res://` instead.
  
  - Replacing scenes requires loading the scene, which loads any resources present in the scene. Make sure any resources present (e.g. scripts) are modified before the scene is replaced.

- `loadDLC()` loads any DLC that the user may have installed.
  
  - This **must** be used prior to modifying certain scenes, or the DLC will fail to properly load.
  
  - I recommend having it at the start of `_init()`, but feel free to check whether or not your mod requires using it.

- `l()` just prints messages to the `Debug` log.
  
  - `msg:String` is the text you want to print, simple enough.
  
  - `title:String` is there as an indicator of what sent the message, uses your `MOD_NAME` by default.

Additionally, there are a few constants and variants that are required:

- `MOD_PRIORITY` is used to determine mod loading order, with *lower* priorities loading first.

- `MOD_NAME` is the name of your mod, used only for printing to the logs.

- `modPath` is the path to your mod folder, used by the functions I included. It is generated automatically on runtime, so you shouldn't have to touch it.

- `_savedObjects` is required for `replaceScene()` to function, no need to touch it. 

##### Overriding scripts

Modifying vanilla behavior is typically handled be modifying existing scripts, and selectively changing functions relevant to the behavior you want. This is highly preferred over replacing the script entirely, which is significantly more likely to break after updates, and adds unnecessary file bloat. Scripts can even be overridden repeatedly, allowing a mod (or several mods) to modify it several times, with all of the modifications applied to the final script. There are some limitations, to this method however:

- You cannot directly override constants or variants declared outside of a function.
  
  - Variants can still be set on `_ready()`, effectively allowing them to be overridden.
  
  - Constants however, cannot be modified in any way (that I am aware of) without completely replacing a script.

- Built-in functions such as `_ready()`, `_process()`, or `_physics_process()` cannot be overridden.
  
  - New code can still be run in these functions, but the existing code cannot be prevented from running.

To override a script, simply [inherit](https://docs.godotengine.org/en/3.5/tutorials/scripting/gdscript/gdscript_basics.html#inheritance) an existing script and override the functions that you want to modify. Then use `installScriptExtension()` to override the original version. 

##### Replacing Scenes

Scene replacement is the main way of modifying anything that does not rely on scripts, such as Textures, UI, Colliders, Etc. Much like scripts, scenes can be replaced multiple times, combining all of the changes. Scene replacement is very powerful, but there are some things to keep in mind:

- Nodes cannot be *removed* from an inherited scene.
  
  - Node scripts *can* be removed/changed, so replacing them with a simple script that removes the node as soon as it loads is possible. Do not that this still requires loading the node in the first place.
  
  - Nodes can be hidden, which is sufficient for some situations.

- Scene replacement is the main method of modifying [exported variants](https://docs.godotengine.org/en/3.5/tutorials/scripting/gdscript/gdscript_exports.html), instead of overriding them in code.

- If inherited scenes are modified or updated, your modified scene will reflect those changes, making updates less likely to break compatibility.

- During the process of replacing a scene, resources in the scene are loaded, which prevents them from being further modified. Make sure scene replacement is the final step of any modification.
  
  - If inherited scenes inherit from more scenes that are also modified, they must be overridden in sequence to apply all of the changes.

To replace a scene, simple select a scene from the FileSystem, select "New Inherited Scene", and modify it to your liking. You can then use `replaceScene()` to replace the original scene with your own.

##### Adding/Modifying Translations

ΔV uses Godot's [internationalization/translation system](https://docs.godotengine.org/en/3.5/tutorials/i18n/internationalizing_games.html) to easily display text in various languages. This system can also be used to easily modify existing text without directly modifying a scene. Additional translations can also be easily added to support multiple languages in mods. To make adding translations easier, I have included `updateTL()`, which can load translations using the [csv](https://en.wikipedia.org/wiki/Comma-separated_values) format. Vanilla translations can be found for reference in the `res://i18n/` folder of the decompiled project, though do note that vanilla ΔV uses [gettext](https://en.wikipedia.org/wiki/Gettext) (`.po`) instead of csv.

##### Supporting DLC

Modifying some scenes may cause DLC portraits to fail to load. If you find that your mod breaks DLC, you may want to preload the DLC, causing their changes to apply before the mod is loaded. To do this easily, I have included the function `loadDLC()`, which forces the DLC to be loaded when called. Simply call `loadDLC()` at the beginning of your `_init()` to have them loaded before your modifications.

# Deploying And Running The Mod

Deploying mods is extremely simple, just compress the entire mod folder into a `.zip`, no compilation or anything required. If you have not used mods before, you will need to set up the game to use mods as well:

- Navigate to the game's installation directory (the directory containing `Delta-V.exe`)

- Create a new directory called `mods`.

- Place your mod `.zip` in the `mods` folder.

- Run the game with the `--enable-mods` command-line parameter.
  
  - On Steam, this can be done in Properties → General → Launch Options.

# Additional Resources

While it is simply not feasible for this document to cover everything you may want to know about modding ΔV, I can provide some additional links to helpful resources.

- [Community Modding Repository for ΔV: Rings of Saturn](https://gitlab.com/Delta-V-Modding/Mods/-/tree/main)
  
  - Here you will find some old mods for ΔV that can be used as reference, as well as the basic documentation for the ModLoader. 
  
  - Unfortunately this repository is no longer actively maintained, and some of the mods may no longer work.

- [The ΔV Translation Project](https://weblate.kodera.pl/projects/dv/i18n/)
  
  - You can easily search for any text or translation keys you may wish to override.
  
  - Additionally, more help with translating ΔV is always welcome.

- [Godot Engine (3.5) documentation](https://docs.godotengine.org/en/3.5/index.html)
  
  - This is the entire documentation to the Godot game engine. ΔV is currently made with Godot 3.5.3 as of writing this.

- [The ΔV Discord](https://discord.gg/dv)
  
  - There are dedicated modding channels in the discord where I (@Za'krin) would love to help answer any questions you may have.
