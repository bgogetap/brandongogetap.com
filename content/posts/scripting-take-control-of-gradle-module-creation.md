---
title: "Scripting Take Control of Gradle Module Creation"
date: 2022-11-28T13:02:57-08:00
draft: true
---

[This post in video form](https://youtu.be/41lpiezq5v4)

## Intro
Embracing multi-module architecture comes with some overhead around creating new modules...big surprise.

Using the new module wizard in Android Studio quickly becomes an unscalable solution as your project becomes more and more customized. 

## Look at your gradle files
If you take a look at your gradle files, it's likely you have a lot of common setup, ranging from sdk versions that reference project-defined variables, or common dependencies.

All of these are things you have to add after creating a new module from the wizard.

Let's fix that.

## Python script
For my projects I have a python script and a folder of template files for each type of module. You can do it however works best for you, but for myself, I have a dedicated script for each type (e.g. Android or Kotlin Multiplatform). 

I've explored using a single script with different arguments to define the module type, but it's just simpler, in my opinion, to have a script for each type (with minor tweaks supported via arguments).

### Template Walkthrough
Let's go over the templates and script for an example Android library module generator.

### build.gradle.kts
I have SDK versions defined in a buildSrc module.

I wrap sections related to compose in markers that I can filter for in the script. This allows me to add an argument when running the script to optionally add compose-related dependencies and setup sections. 

Then in the dependencies section I have my most common dependencies. Here I erred towards requiring the removal of unneeded dependencies after creation, rather than adding more script arguments to make the generation ultra-granular. This is a balance you'll have to strike for yourself. I personally find removing lines here and there to be low enough friction (especially when compared to needing to *add* lines). For example, not all of my library modules require Dagger.

### other templates
The other templates for my current project aren't anything special. A super simple .gitignore file, and default proguard rules file.

The manifest is generated completely in the script, since it's a one-liner, so there is no template for that, though that is something you could add.

## Script Walkthrough
Now let's jump to the script. 

I use ArgParse to define the script args, which for now are just the module name (with nesting possible via `.` separator) and whether or not the module uses Compose.

We define the root for this script, which is this case in the android folder of my multiplatform project.

Next we have values for the template file locations. 

Before we look at the functions, let's scroll down to the bottom and see the order of execution.

We get the arguments via argparse.

pull out the module name, then split it on `.` in case this is a nested module

Now we create directories. Please forgive the extremely helpful, lone comment I apparently though was needed in this file.

Then we can start adding the files.

The proguard file is a simple copy of our template

The manifest file is a copy of the defined string we have in our script. Though we also have to replace the placeholder with our new module's path.

Next we add the gradle file. Here we also pass in whether or not we're using Compose, and in our function we detect compose section start/end and include (or not) those lines.

Let's check out what modules generated with this end up looking like

## Running the script

Let's create a couple modules to demonstrate the different things this script can do.

First, we'll create one that doesn't use compose. User prefs could be a good use case.
- Call this module "prefs"

Then one that uses compose. We'll make this a "feature" module, and nest it in the feature directory. This will be for the "home" screen of our app.
- Call this module "home"

As we can see, the expected directories were created, and running a gradle sync automatically includes these new modules in our project using the [[Gradle Auto-Add Module]] strategy that I've discussed in another video.

## Summary
There isn't anything groundbreaking here, but I have found it to be incredibly valuable, especially as I've come to embrace the very granular modularity that a large monorepo, which I deal with in my current company, requires.