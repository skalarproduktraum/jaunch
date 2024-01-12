*Jaunch: Launch Java **Your** Way!™*

Jaunch is a native launcher for applications that run on a Java Virtual Machine (JVM).

## Quick Start

* **Linux or macOS**
  ```shell
  ./build.sh my-app /path/to/MyApp
  ```

* **Windows:** Install [Scoop](https://scoop.sh/) first, then:
  ```powershell
  scoop install mingw
  sh build.sh my-app \path\to\MyApp
  ```

Where `my-app` is what the final native launcher executable should be named,
and `/path/to/MyApp` is where you want the Jaunch launcher files to be copied.

The build script will:

1. Build the native C code, generating a binary named `my-app` (`my-app.exe` on Windows).

2. Build the Kotlin code, generating a binary named `jaunch`/`jaunch.exe`.

3. Copy the needed files to the specified application path, including:
   * The two native binaries (1) and (2);
   * Two TOML configuration files, `jaunch.toml` and `my-app.toml`;
   * The `Props.class` helper program.

Then edit the `my-app.toml` to match your application's requirements.
(You can also edit `jaunch.toml`, although doing so is typically unnecessary.)

Then try running `my-app` and watch the fireworks. If it doesn't work,
try `my-app --debug`, which will show what's happening under the hood.

## Why Jaunch

Do you have a desktop application that runs on the Java Virtual Machine? You probably
want a friendly native launcher, for example a .exe file in the case of Windows, that
launches your program, right?

To accomplish that, you can use
[jpackage](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jpackage.html) to
generate one for every platform you want to support&mdash;typically Linux, macOS, and
Windows. The jpackage tool is part of the Java Development Kit (JDK), so you might think
there is no need for another Java native launcher.

But jpackage is inflexible:

* The jpackage tool mandates a specific directory structure for your application,
  e.g. putting all JAR libraries in the `lib/app` folder, and the bundled Java
  runtime into `lib/runtime`. As far as I know, you can't use a different structure.

* The native launchers generated by jpackage do not allow users to pass custom arguments
  to the JVM (["users can't provide JVM options to the application"][1]). All arguments
  are handed verbatim to your Java application's main method. So if your users want to
  e.g. increase the maximum heap size by passing -Xmx20g, they must edit the jpackage
  .cfg file located in the application's `/app` directory.

* The jpackage tool also provides no way to support options like `--debugger=8000`
  which are transformed into JVM arguments like
  `-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=localhost:8000`.
  To simulate that sort of thing with jpackage, you'd need to implement an option in
  your application's interface like "restart application in debug mode" with a port
  and a "suspend" checkbox. And you'd have to decide whether to reset that every time
  your application launches, or else require the user to go back into the options and
  disable the debug mode again for next time when they are finished.

* jpackage apparently no longer supports generating an application without a bundled
  Java. It did once, back when it was the "JavaFX packager", but based on Internet
  searches and the docs, it seems they removed this feature. So you cannot have a
  "portable application" version of your program, runnable from a USB stick across
  multiple platforms, as long as the computer has a system installation of Java.

* jpackage has a feature to generate multiple launchers, each with its own arguments and
  main class, which is nice. But they end up as separate executables, so you cannot have
  an option like `fizzbuzz --update` that runs a different main class. (It would be
  doable on the Java side by having a main class that always gets run, which then
  delegates to another main class, but it's a hassle.) Whereas the design of Jaunch
  enables a `--main-class` option so that `fizzbuzz --main-class org.example.AltClass`
  will run `AltClass` rather than FizzBuzz's default main class.

* On Windows, a jpackage launcher must be either a console application, or a non-console
  application. It cannot be both, contingent on how the user invokes it. Whereas Jaunch
  connects to the existing console when there is one, but does not create a new one.

All of that said, jpackage is a very nice tool, and if it works for you, use it! But if you
want more flexibility&mdash;if you want to launch Java ***Your** Way*&mdash;then read on.

## Design Goals

### Run Java in the same process as the launcher

* Be a good citizen of our native environment.
* Integrate properly with application docks, system trays, and taskbars.

### Discover Java installations already on the system

* Recognize system-wide installations as well as bundled Java runtimes.
* Link to the best libjvm on demand; do not invoke `java` in a separate process.
* Search beneath specified JVM directory roots.
* Define rules for deciding which installations meet application requirements.
* If no appropriate Java installation is found, show an informative error message.

### Support runtime customization of Java launch parameters

* Customize at runtime which arguments are passed to the JVM.
* Customize at runtime which arguments are passed to the main class.
* Customize at runtime which Java main class is run.

### Customize launcher behavior without recompiling native binaries

* Editable TOML files let users customize launcher behavior,
  e.g. adding shortcuts for common command line operations.

### Keep as much code in a high-level language as possible

* Jaunch came into being to replace an aging launcher written as 5000+ lines of C code.
  The design centers around minimizing the size and complexity of needed C code,
  in favor of most code being written in a more maintainable high-level language.
* On the off-chance that the TOML-based configuration is not flexible enough for your
  application's needs, your next layer of customization is the Kotlin codebase, not C.

## Architecture

TODO - describe Jaunch's architecture here.

- jaunch.exe is the Kotlin program. It does not need to be modified.
- The native launcher (built from jaunch.c) should be named whatever you want. E.g. fiji.exe.
- fiji.toml is the configuration that jaunch.exe uses to decide how to behave.
  - When fiji.exe invokes jaunch.exe, it passes `fiji` to jaunch.exe.
- In this way, there can be multiple different launchers in the same directory that all use the same jaunch.exe.

Discover available Javas from:
- Subfolders of the application (i.e. bundled Java).
- Known tool-specific installation locations.
  - sdkman
  - install-jdk
  - cjdk
  - conda
  - brew
  - scoop
  This is done in a general way by having a list of directories in the default TOML configuration, which can be extended by specific applications as desired.

.. more to come ...

## Alternatives

As so often in technology, there are so many. And yet nothing that does what this program does!

### Executable JAR file

* Pros:
  * Simple: double-click the JAR.
* Cons:
  * Needs OpenJDK already installed and registered to handle the .jar extension.
  * Encourages creation of uber-JARs over modular applications.
  * Does not integrate well with native OS application mechanisms.

### jpackage

* Pros:
  * [Official tooling](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jpackage.html) supported by the OpenJDK project.
* Cons:
  * Imposes its opinions on the application directory structure (e.g. jars must go in `libs` folder).
  * Standalone executable launcher does not support passing JVM arguments (e.g. `-Xmx5g` to override max heap size).
  * Always bundles a Java runtime, rather than discovering existing Java installations.
* Example:
  * The [QuPath](https://qupath.readthedocs.io/) project uses jpackage for its launcher.
    * QuPath provides a configuration dialog to modify the Java maximum heap size, which it handles by editing the jpackage .cfg file on the user's behalf.

### Call `java` in its own separate process

E.g., via shell scripts such as `Contents/MacOS/JavaApplicationStub`.

* Pros:
  * Very flexible and easy to code.
* Cons:
  * Needs OpenJDK already installed and available on the system `PATH`, and/or pointed at by `JAVA_HOME`, and/or known to `/usr/libexec/java_home` on macOS, `/usr/bin/update-java-alternatives` on Linux, etc. In a nutshell: you are doing your own discovery of OpenJDK installations.
* Example:
  * The [Icy](https://icy.bioimageanalysis.org/) project uses shell script launchers on macOS and Linux, and a mystery meat (AFAICT) .exe launcher on Windows.
    * For Linux, the `java` on the system path is used.
    * For macOS, the `java` given by `/usr/libexec/java_home -v 1.8` is used.
    * In both cases, no arguments can be passed to the program (neither to the JVM nor to the Icy application).

### Lean on command-line tools

E.g. [**SDKMAN!**](https://sdkman.io/), [cjdk](https://github.com/cachedjdk/cjdk), [install-jdk](https://github.com/jyksnw/install-jdk), [jgo](https://github.com/scijava/jgo).

* Pros:
  * Leave it to dedicated external code to install and manage your JDKs.
* Cons:
  * Unfriendly to require non-technical users to run terminal commands to launch a GUI-based application.

### Use a general-purpose Java launcher

[install4j](https://www.ej-technologies.com/products/install4j/overview.html) by ej Technologies.
* You can pass parameters to the JVM at runtime [via the `-J` argument prefix](https://stackoverflow.com/a/63318626/1207769).
* Closed source.

[launch4j](https://launch4j.sourceforge.net/)
* Must choose console or GUI a priori for Windows flavor.
* Still hosted on SourceForge.
* Can customize JVM options at runtime, but only by editing the application's `.l4j.ini` file.

[WinRun4J](https://github.com/poidasmith/winrun4j)
* Windows only.
* [Unmaintained](https://github.com/poidasmith/winrun4j/issues/102) since 2018.

### Build your own native launcher

[JavaCall.jl](https://github.com/JuliaInterop/JavaCall.jl)
* Written in Julia, permissively licensed.
* Provides a general API for working with the JVM from Julia code.
* Would need to build a native launcher on top of it.
* Does not work on macOS anymore with Julia 1.6.3+.

[hfhbd/jniTest](https://github.com/hfhbd/jniTest)
* Written in Kotlin (Native/KMP application).
* Proof of concept for loading libjvm to launch Java code from a Kotlin program.
* Built binary is 518K.
* Does not use `dlopen`/`dlsym`, but rather links to libjvm. See my [post on Kotlin Discuss](https://discuss.kotlinlang.org/t/27756).
* Also links to libpthread, unlike other native launchers on this list.

#### Examples

[ImageJ Launcher](https://github.com/imagej/imagej-launcher)
* Written in C.
* Built binary is 91K.
* Supports custom runtime arguments to both ImageJ/Fiji and the JVM.
* Sometimes needs to re-exec with `execvp`, either itself with changes to environment variables, or the system Java.

[ijp-imagej-launcher](https://github.com/ij-plugins/ijp-imagej-launcher)
* Written in Scala.
* Built binary is 7.8M.
* Links to libstdc++, unlike other native launchers on this list.
* Calls `java` in a separate process.

[Why](https://github.com/AstroImageJ/Why) (AstroImageJ)
* Written in Rust.
* Uses [jni-rs](https://github.com/jni-rs/jni-rs) crate, which leans on the intelligent Java discovery of [java-locator](https://crates.io/crates/java-locator).
* Built binary is 51M.
* Minimal dynamic library dependencies.
* Project clearly targets Windows only; I had to modify the `Cargo.toml` and `file_handler.rs` in order to compile it for Linux.
* No stated license.

------------------------------------------------------------------------------

[1]: https://docs.oracle.com/en/java/javase/20/jpackage/support-application-features.html#GUID-B350A40C-3239-4C20-B5E3-79B4F81F1CEE
