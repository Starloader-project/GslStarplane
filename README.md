# GslStarplane: Starplane for Gradle

GslStarplane is the gradle port of the starplane buildsystem originally written
for use in combination with Brachyura. Starplane lays the foundation of modern
galimulator modding.

## Including the plugin

**WARNING:** When publishing your code online with this plugin enabled, you
should make extra sure to NOT include the `build` directory. GslStarplane
pushes deobfuscated jars which have a copyright on them to it.

GslStarplane functions as a gradle plugin, so it first needs to be applied on a
gradle project in order to work. Under groovy you can do it by adding
`id 'gsl-starplane' version '0.1.0'` to the plugins block in the `build.gradle`
file. The full plugins block will thus look something like follows:

```groovy
plugins {
    id 'java'
    id 'java-library'
    id 'maven-publish'
    id 'gsl-starplane' version '0.1.0'
}
```

However, as GslStarplane is not located in the default repositories and
depends on external projects, you also need to add a few maven repos to the
pluginManagement block of the `settings.gradle` file. In full the additions
probably look like that:

```groovy
pluginManagement {
    repositories {
        gradlePluginPortal()
//      mavenLocal()   // Uncomment this line if you wish to debug the plugin
        mavenCentral()
        maven {
            name = 'Geolykt'
            url = 'https://geolykt.de/maven/'
        }
        maven {
            name 'fabric'
            url 'https://maven.fabricmc.net/'
        }
    }
}
```

And that is it! Unfortunately you won't be able to use the plugin for much
with such a setup. So continue reading!

## Specifing reversible access setters

**WARNING:** Reversible access setters are as dangerous as they are simple to
use. Changing access of a method could make methods that normally would be
independent of each other to override each other.

Reversible access setters can optionally be declared in the `build.gradle`
as follows:

```groovy
starplane {
    withRAS(rootProject.file("src/main/resources/starloader-api.ras"))
}
```

Starplane will automatically apply the contents of the reversible access
setter on the galimulator jar and will remap all \*.ras file during the
`remap` task.

At the moment, only one reversible access setter can be set at a time
and transitive reversible access setters are ignored.

**Declaration of reversible access setters to starplane is independent
from the Starloader-launcher declaration of reversible access setters!**

## The `galimulatorDependencies` configuration

Starplane automatically strips dependencies it can find from the galimulator
jar. This may sound unintuitive but has the benefit of reducing the amount
of classes that need to be decompiled and also provides (if configured
properly) javadocs and sources in the IDE, leading to a more comfortable
development experience.

The dependencies that need to be stripped can be configured through the
`galimulatorDependencies` configuration, however it is advised that the
configuration is not touched. However, by default dependencies which are in the
`galimulatorDependencies` configuration are not on the compile classpath.
To change this, `compileOnlyApi` can be made to extend from
`galimulatorDependencies`. In practice this can be done by inserting

```groovy
configurations {
    compileOnlyApi.extendsFrom(galimulatorDependencies)
}
```

in the project buildscript.

## The `remapJar` task

The `remapJar` task remaps all references of deobfuscated members to use
the obfuscated names instead. It produces the jar that can be used outside
the development environment and which you can freely distribute
(should there not be other limitations).

The `remapJar` task extends the `jar` task, which means that it can
be configured similar to `jar`. However, unlike the `jar` task
you probably want to define the `from` inputs.

Note: It is recommended to set the `archiveClassifier` of the
remapped jar as otherwise the `jar` task cannot be cached.

Including a jar will insert the jar in the built jar root.
This is probably not intended behaviour for you, so you'd need to decompress
is beforehand. Alternatively, the `fromJar` method can be used to include
jars. Furthermore, `fromJar` implicitly adds dependencies on Tasks if the input
is a task. The below example shows just that.

The recommended configuration of the `remapJar` task is follows:

```groovy
remapJar {
    archiveClassifier = 'remapped'
    fromJar jar
}
```

Sometimes muscle memory gets the better of you and you are still
acustomed to using `build` to create jars. Unfortunately, by default, `build`
does not create a remapped jar. In order for the `build` task to also remap,
one can make `build` depend on `remapJar`. In practice this would look as
follows:

```groovy
build {
    dependsOn remapJar
}
```

## Defining the mods in the development environment

The mods that are run in the dev env (which is started through the `runMods`
task that will be talked about later on) can be selected by configuring the
`deployMods` task. If you mod does not depend on any other mods, nothing needs to
be done. However if you want to depend on third party mods, an approach such
as the one below can be used: 

```groovy
deployMods {
    from "libs/mymod.jar"
}
```

Aside from `String` (which is used in the above example) other types can be used.
A complete list of supported types is as follows:

 - `PublishArtifact`
 - `AbstractArchiveTask` tasks (includes `Jar` tasks)
 - `DefaultAdhocSoftwareComponent` components
 - `Configuration`s
 - `String`, `File`, `Path` and all other types that can be converted to `File`
   as per [Project#file](https://docs.gradle.org/8.1/javadoc/org/gradle/api/Project.html#file-java.lang.Object-).

Multiple files can be referenced, however only files that are mods (as in
Starloader-launcher extensions) will get copied into the extension directory.
All mods with the same id/name that already exist in the extension directory
will get removed in order to avoid duplicates.

During the copy process the starplane annotations will get evaluated.

**WARNING:** Removing a mod from the `deployMods` list will not remove them
from the extensions folder yet. This may get changed should there be sufficent
momentuum.

Reversible access setters from other mods ("transitive reversible access
setters") are not applied at compile-time, but are applied as usual at
runtime.

In order add external mods via maven, the following approach can be used:

```groovy
// Note: The order of the "configurations" block matters - it should be over the "deployMods" block.
configurations {
    dependencyMods // Define the "dependencyMods" configuration so it can be used
    compileOnlyApi.extendsFrom(dependencyMods) // Also add all deployed mods to the compile classpath
}

deployMods {
    // Tell starplane to deploy all mods from the "dependencyMods" configuration
    from configurations["dependencyMods"]
}

dependencies {
    // Add the mod to the configuration (you can also add several mods to the same configuration)
    dependencyMods "de.geolykt:starloader-api:2.0.0-SNAPSHOT"
}
```

## Running the development environment

The dev env can be run through the `runMods` task. The development environment
allows you to test your mod quickly while still using deobfuscated mappings.

The `runMods` task is a `JavaExec` task, which means it can be configured as one
(if you need to add JVM Arguments), however in default circumstances no further
changes need to be applied.

In order for the dev env to work, the `deployMods` task needs to be correctly
configured.

## The `genEclipseRuns` task

While the `runMods` task works to execute the development environment, it does
not allow to easily debug the environment with the aid of a debugger. While
setting the `debug` option of the `runMods` task to true (as allowed by
`JavaExec`) does allow to attach a debugger to the environment, IDEs such as
Eclipse struggle with finding the sources of the classes. This can make
debugging difficult. To counteract this, the `genEclipseRuns` task generates
the `runMods.launch` file that can be used to execute the development
environment right within your IDE with all the extras your IDE provides.

**NOTE:** In most cases the generated \*.launch files won't work as necessary
tasks (such as negating AWs) aren't run.
To remedy this issue, improvements need to be done on the Starloader-launcher.
Such improvements would also mean the end of whacky workarounds such as
gslStarplane generating runtime-access and compile-time access jars.
With negated AWs only one slim compile-time access jar would be needed.

## Selecting the Mod loader (the `devRuntime` configuration)

As of now, only the Starloader launcher can be used as a mod loader (though
in theory modloaders that work as javaagents can easily be added). The version
used depends on the contents of the devRuntime configuration classpath
- so the starloader launcher needs to be either present on the runtime classpath
or be explicitly declared as being part of the devRuntime configuration in order
for the dev env to work. The Starloader Launcher can thus be declared as follows:

```groovy
dependencies {
    // [...]
    devRuntime "de.geolykt.starloader:launcher:4.0.0-20230520"
    // [...]
}
```

The devRuntime configuration extends from the `runtimeClasspath` configuration,
so elements you added through `runtimeOnly` or similar will be available.
However as this may has consequences on classloading that may be removed
in the future (it is plausible that all runtime elements will be available to us
anyways due to shading).

It is generally not adviseable to add mods through the devRuntime, instead the
deployMods should be configured accordingly. Failure to understand this may
result in mods not properly loading or other classloading issues.

## Selecting the Mappings

At the moment only spStarmap ontop of slintermediary can be used.
In the (far) future other variants of deobfuscation mappings (such as mmStarmap)
may get supported.

## Decompilation

gslStarplane decompiles Galimulator with Quiltflower, a Fernflower-based
decompiler. The dependencies added on the decompilation classpath are
controlled by the `galimulatorDependencies`. Furthermore, gslStarplane always
decompiles the stripped galimulator jar with compile-time access.

The original line mappings are visible in the decompiled output,
but starplane automatically changes the line mappings of the runtime and
the stripped compile-time jars to reflect the line mappings of the decompiled
output.

**NOTE**: I am aware that QuiltMC (the organisation behind Quiltflower)
had a serious internal disruption starting from the 20th April of 2023. Due
to the organisation seemingly not being aware of the importance of good wording
(please: be aware that words are the most important thing ever if you intend
to intend to depos someone. Using wording that can only be described as counter-
productive is stupid), I have been banned on Quilt's toolchain discord. I thus
deem it likely that I have been completely banned from the entirety of the quilt
project. The current strategy is to just wait and hope that they forget that I
am banned. Should that not work, Quiltflower will probably get forked by us (or
we use a fork from an organisation that suffered a similar fate).

## Eclipse external null annotations

In order to provide full parity over the old starplane on slbrachyura system
gslStarplane supports the feature you'd expect the least: Eclipse external null
annotations. EEAs can be added like so:

```groovy
plugins {
    // [...]
    id 'eclipse'
}

starplane {
    eclipseEEA = rootProject.file("src/eclipse-eea")
}
```

and will be applied on all gradle classpath elements.


Note: the backend code within gslStarplane is basically equal to

```groovy
apply plugin: 'eclipse'

eclipse {
    classpath {
      containers 'org.eclipse.buildship.core.gradleclasspathcontainer'  
      file {
            whenMerged {
                def source = entries.find { it.path == 'org.eclipse.buildship.core.gradleclasspathcontainer' }
                source.entryAttributes['annotationpath'] = "src/eclipse-eea"
            }
        }
    }
}
```

## Roadmap

Currently it is planned to offer Starplane as Eclipse and IntelliJ plugins,
however due to the required effort needed to learn their plugin APIs this
will only happen in the far future - if ever.

A Maven integration is unlikely due to it apparently lacking the ability
to add dependencies programatically without touching the POM.
