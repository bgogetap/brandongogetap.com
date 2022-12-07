---
title: "Gradle Auto Include Modules"
summary: "Update your settings.gradle.kts to auto-include all modules in your project"
date: 2022-11-28T12:00:00-08:00
---

[This post in video form](https://youtu.be/msCs86Puh6c)

## Intro
As someone who has become very accustomed to granular modules, I found having to add each module to the `settings.gradle` to be a mild annoyance, and figured there would be a simple way to get around that.

This post essentially boils down to a snippet that you can add to your `settings.gradle.kts` file to auto-include all gradle modules present in your project.

{{< gist bgogetap c1610fbb4f83e2953b541b838c47e62a >}}

The rest of the words below just dig into how that works and why you may want it.
______

## Creating a module with the wizard
The standard way we add modules is via the built-in wizard in Android Studio/IntelliJ. This is pretty convenient, as it takes care of updating the `settings.gradle` file by adding the `include(...)` statement for your new module.

However, in larger projects, and likely many of your own side projects, it is possible that you are not creating modules via the integrated New Module wizard, but rather with scripts that you've created that take care of some specific logic you define for your company or personal project. 

## Creating a module with a script
Using a script to create new modules is a great way to save yourself and/or your team time by removing boilerplate.

We aren't going to dig into a particular script in this post, though I will have a post soon that goes through the one I use for my personal projects.

_____

After the module is created with the script, running Gradle sync will result in no changes, even though we have a new `build.gradle` file. This is obvious at this point, and it's because we didn't add the `include(...)` statement for the new module.

For an app with a few modules, this isn't a big deal. But as your nesting gets deeper, like it would in a mono-repo, having to add includes that potentially have three or four or even more parts to the module name is less than ideal.

> **_NOTE:_** You might be thinking "I can just have my script update the `settings.gradle` file itself". That's true, and is another option to accomplish this same behavior. I personally find explicit `include(...)` statements to be fragile, and would rather a dynamic, never-failing method be used, assuming the downsides are minimal/non-existent.

### Using the snippet
By adding the code above to your `settings.gradle.kts` file, you'll never have to manually write an `include(...)` statement again.

Let's walk through it:

### Setup
```kotlin
val modules = mutableSetOf<String>()
val ignoredDirectories = setOf("build", "buildSrc", "gradle", ".idea")
```

We start with a mutable Set of Strings that represents each target that needs to be included.

We can save some time by skipping directories that we know won't have gradle modules.

### Find all modules
```kotlin
fun fillModules(files: List<File>, targetPath: String) {
    files.forEach { file ->
        if (file.isFile) {
            if (file.name == "build.gradle" || file.name == "build.gradle.kts") {
                modules.add(targetPath)
            }
        } else {
            if (ignoredDirectories.contains(file.name)) {
                return@forEach
            }
            val updatedParentPath = "$targetPath:${file.name}"
            val childFiles =
                file.listFiles()?.filter { it.isDirectory || it.name.contains(".gradle") }
            if (childFiles != null) {
                fillModules(childFiles, updatedParentPath)
            }
        }
    }
}
```
The `fillModules` function takes in a List of Files, and the `targetPath`. This target path will change as we walk deeper into the project's directories. This way, we can support nested modules (e.g. `android:feature:home`).

If we find a `build.gradle[.kts]` file, we know we're in a module and can add the current target to our Set.

Otherwise we keep walking down the file tree.

### Execute
```kotlin
val projectDirectories = checkNotNull(
    rootProject.projectDir.listFiles()
        ?.filter { it.isDirectory }
) { "No directories in project" }

fillModules(
    files = projectDirectories, targetPath = ""
)
modules.forEach { include(it) }
```

Eventually we've found all targets that have a `build.gradle` file, and we can simply loop through them and `include` each one.

After we create some new library modules with our script, we can run gradle sync and they will be part of our project.

We can even move them into a nested directory, run sync again, and the module is updated in our project as expected.

## Potential downsides

### Performance / Build time impact
When testing performance impact on a project with 86 modules, I noticed no significant downside to this approach. Finding the modules took a few hundred milliseconds on my Intel Macbook Pro (the range was actually quite wide, depending on my computer's mood, though never exceeded 800ms).

Using this approach also has no impact, from what I could tell, on clean or incremental builds, aside from the time it takes to find the modules.

### Selective `include`?
Does this approach limit selective `include`s? I don't see how, or rather, the same logic that is definining a subset of modules to include in a monorepo could be adapted here. However if you're at that stage, I assume there is already something else driving this process.

## Summary
This may seem trivial, but as someone who has a deep addiction to side projects and granular modularity, this is one of my favorite, small quality-of-life changes, coupled with the script to generate modules (which I will have a post on soon).

## Follow Up: Magic == Danger
I asked about this strategy on Mastodon ([check out thread here](https://mastodon.social/@bgogetap/109424092373947165)) and some valid points were brought up about potential issues. 

If you're using git submodules, this strategy would end up including Gradle modules from those projects that you likely didn't intend to. In the end, this is a case of potentially overly clever logic leading to silent issues that may be hard to debug.

It's something to keep in mind, and this auto-include strategy is clearly not the right choice for every project. I will personally continue using it on my personal projects for a couple reasons:
- During project ramp-up, I do a lot of refactoring and module creation, and this strategy takes out a small pain-point of that process.
- Converting to explicit includes is as simple as adding a `println("include(it)")` line in the loop, copying that output, and pasting it in your `settings.gradle.kts` file while deleting/commenting out the auto-include code. In other words, changing from auto-include to explicit-include (maybe once the project is mature) is a dead simple process, so there is no major downside to starting with auto-include.