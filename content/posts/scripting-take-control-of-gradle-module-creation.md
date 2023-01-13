---
title: "Scripting Take Control of Gradle Module Creation"
date: 2023-01-13T12:00:00-08:00
---

[This post in video form](https://youtu.be/41lpiezq5v4)

This post will be referencing the specific script and templates used in [the KMM GitHub Browser project](https://github.com/bgogetap/GitHub-Browser-KMM/blob/main/android/scripts/gen_module.py)

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

**build.gradle.kts**
{{< gist bgogetap cf2c448945cbf9f915f4c039fc94edd1 "lib_module_template.gradle.kts" >}}

The goal is for minimal required changes after generation. To accomplish that:

SDK versions defined in a buildSrc module.

Sections related to Compose are wrapped markers that can be detected in the script. This allows the opportunity to use an optional script argument add compose-related dependencies and setup sections when required. 

Finally, in the dependencies section, we put the most common dependencies. Here I tend to lean towards *requiring the removal of unneeded dependencies after creation, rather than adding more script arguments to make the generation ultra-granular*. This is a balance you'll have to strike for yourself. I personally find removing lines here and there to be low enough friction (especially when compared to needing to *add* lines). For example, not all of the library modules in the project that uses this script require Dagger.

**other templates**

The other templates for my current project aren't anything special. A super simple .gitignore file, and default proguard rules file.

The manifest is generated completely in the script, since it's a one-liner, so there is no template for that, though that is something you could add.

## Script Walkthrough
{{< gist bgogetap cf2c448945cbf9f915f4c039fc94edd1 "generate-android-module.py" >}}

ArgParse is used to define the script args, which for now are just the module name (with nesting possible via `.` separator) and whether or not the module uses Compose.

### Constants
```python
PROJECT_ROOT = os.path.realpath(os.path.dirname(__file__)).rsplit('/', 1)[0]

MANIFEST_TEMPLATE = '''<manifest package="dev.neverstoplearning.githubbrowser.{module}"/>
'''

GRADLE_TEMPLATE_FILE = PROJECT_ROOT + "/templates/lib_module.gradle.kts"
PROGUARD_TEMPLATE_FILE = PROJECT_ROOT + "/templates/proguard-rules.pro"
GITIGNORE_TEMPLATE_FILE = PROJECT_ROOT + "/templates/.gitignore"
```
We define the root for this script, which should be one directory up from where this file resides.

Next we have values for the template file locations.

### Execution
Before we look at the functions, let's skip to the bottom and see the order of execution.

```python
args = parser.parse_args()

module_name = args.name

dirs_to_create = module_name.split(".")
```

We get the arguments via argparse.

pull out the module name, then split it on `.` in case this is a nested module

```python
module_dir = add_module_dirs(dirs_to_create)

add_proguard_file(module_dir)
add_manifest(module_dir)
add_gradle_file(module_dir, args.use_compose)
add_gitignore_file(module_dir)
```

Now we create directories and start adding the files.

The proguard file is a simple copy of our template

```python
def add_proguard_file(dir):
    proguard_file = add_file(dir, "proguard-rules.pro")
    with open(PROGUARD_TEMPLATE_FILE) as template_file:
        for line in template_file:
            proguard_file.write(line)
```

The manifest file is a copy of the defined string we have in our script. Though we also have to replace the placeholder with our new module's path.
```python
def add_manifest(dir):
    main_src_dir = dir + "/src/main/"
    os.makedirs(main_src_dir, exist_ok=True)
    manifest_file = add_file(main_src_dir, "AndroidManifest.xml")
    manifest_value = MANIFEST_TEMPLATE.replace("{module}", args.name)
    manifest_file.write(manifest_value)
```

Next we add the gradle file. Here we also pass in whether or not we're using Compose, and in our function we detect compose section start/end and include (or not) those lines.
```python
def add_gradle_file(dir, use_compose):
    gradle_file = add_file(dir, "build.gradle.kts")
    with open(GRADLE_TEMPLATE_FILE) as template_file:
        compose_section_started = False
        for line in template_file:
            if "{{compose-start}}" in line:
                compose_section_started = True
                continue
            if "{{compose-end}}" in line:
                compose_section_started = False
                continue
            if compose_section_started and not use_compose:
                continue
            gradle_file.write(line)
```

Finally, the source directories are created
```python
package_dirs_to_create = ["dev", "neverstoplearning", "githubbrowser"] + dirs_to_create
add_src_dirs(module_dir, package_dirs_to_create, "androidTest")
add_src_dirs(module_dir, package_dirs_to_create, "main")
add_src_dirs(module_dir, package_dirs_to_create, "test")

add_res_dirs(module_dir)
```

## Summary
There isn't anything groundbreaking here, but I have found it to be incredibly valuable, especially as I've come to embrace the very granular modularity that a large monorepo, which I deal with in my current company, requires.