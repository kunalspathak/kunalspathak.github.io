---
layout: post
title: Doskey in Windows
subtitle: A hidden MS-DOS "alias" utility
tags: [work, tools]
comments: true
---

## Introduction

Ever since I came to know about the `doskey` command in MS-DOS, I could barely stop myself using it. Not many Windows developers are familiar with this command, but this is a very powerful command to have in the arsenal of any Windows developer who frequently works on command line. If you type `doskey /?`, here is what you will see:

```

Edits command lines, recalls Windows commands, and creates macros.

DOSKEY [/REINSTALL] [/LISTSIZE=size] [/MACROS[:ALL | :exename]]
  [/HISTORY] [/INSERT | /OVERSTRIKE] [/EXENAME=exename] [/MACROFILE=filename]
  [macroname=[text]]
vvv 
  /REINSTALL          Installs a new copy of Doskey.
  /LISTSIZE=size      Sets size of command history buffer.
  /MACROS             Displays all Doskey macros.
  /MACROS:ALL         Displays all Doskey macros for all executables which have
                      Doskey macros.
  /MACROS:exename     Displays all Doskey macros for the given executable.
  /HISTORY            Displays all commands stored in memory.
  /INSERT             Specifies that new text you type is inserted in old text.
  /OVERSTRIKE         Specifies that new text overwrites old text.
  /EXENAME=exename    Specifies the executable.
  /MACROFILE=filename Specifies a file of macros to install.
  macroname           Specifies a name for a macro you create.
  text                Specifies commands you want to record.

UP and DOWN ARROWS recall commands; ESC clears command line; F7 displays
command history; ALT+F7 clears command history; F8 searches command
history; F9 selects a command by number; ALT+F10 clears macro definitions.

The following are some special codes in Doskey macro definitions:
$T     Command separator.  Allows multiple commands in a macro.
$1-$9  Batch parameters.  Equivalent to %1-%9 in batch programs.
$*     Symbol replaced by everything following macro name on command line.
```

I will not be explaining all the features of this command, but will elaborate on very important feature called `MACROS`. This command lets you create a shorthand alias for your command. Once you create an alias, you can trigger that command using the alias. The alias can also take the command line parameters that you would have otherwise passed to the original command. 

## Examples

Let's say that you frequently execute command `findstr /sin <search string>` to search a string in a directory, you could set an alias for that command:

```
C:\>doskey find=findstr /sin $*

C:\>find abc

<<results of findstr>>
```

Here, I have created an alias `find` and mapped it to the DOS command I want to execute i.e. `findstr /sin` in this case. The parameter `abc` to the alias is forwarded to the DOS command that is mapped to that alias.

However, I find `doskey` more useful in triggering various development commands too like build, run, copying files, opening debugger, etc. To give an example, the command to build dotnet's [coreclr](https://github.com/dotnet/runtime/blob/master/docs/workflow/building/coreclr/README.md) is `<repo_root>\build.cmd -subset clr -configuration checked -arch x64`. I have mapped `bclr` alias to this command:

```
C:\runtime>doskey bclr=pushd .&C:\runtime\build.cmd -subset clr -configuration checked -arch x64&popd

D:\some_dir>bclr

<<triggers build for x64 checked>>
```

Now imagine you need to trigger the `bclr` command for other architectures and configurations. You could pass the arguments in the form of `$1`, `$2` and so forth to the alias. For that, you need to configure alias to accept parameter.

```
D:\some_dir>doskey bclr=pushd .&C:\runtime\build.cmd -subset clr -configuration $2 -arch $1&popd

D:\some_dir>bclr arm64 checked

<<trigger build for arm64 checked>>
```
 
<p/>

## My favorite aliases

Here are some of my favorite aliases that I have configured and use frequently.

### Navigation

Below are some of the aliases I set for navigating between various directories while on DOS command line.

```
C:\>doskey ..=..\$*
C:\>doskey ...=..\..\$*
C:\>doskey mydir=cd /d C:\this\is\mydir

C:\deep\down\dir\structure>..
C:\deep\down\dir>
C:\deep\down\dir>..anotherdir
C:\deep\down\anotherdir>
C:\deep\down\anotherdir>mydir
C:\this\is\mydir>
```

<p/>

### Development commands

You can set alias to perform your development activities like building, opening your favorite IDE or debugger.

To open your project using appropriate version of Visual Studio:

```
C:\>doskey worksln="C:\Program Files (x86)\Microsoft Visual Studio\2019\Common7\IDE\devenv.exe" C:\git\work\somedir\work.sln
```

To trigger build command:

```
C:\>doskey buildall=pushd C:\git\work&buildir\build.cmd x64 checked&popd 
```

To configure windbg with favorite settings:

```
C:\>doskey dbg=C:\debuggers\windbg.exe -o -y C:\git\work\bin\symboldirPDB -srcpath C:\git\work $*
C:\>dbg work.exe hello world
```

To set the environment variables:

```
C:\>doskey jitasm=set COMPlus_JitDisasm=$*

C:\>jitasm HelloWorld
C:\>set COMPl
COMPlus_JitDisasm=HelloWorld
```

<p/>

### Get/Find/Open aliases

You can also configure some aliases that can help you manipulate the aliases that you have already set.

To list all the aliases set so far:

```
c:\>doskey allkeys=doskey /macros

c:\>allkeys
find=findstr /sin $*
buildall=pushd C:\git\work&buildir\build.cmd x64 checked&popd
bclr=pushd .&C:\runtime\build.cmd -subset clr -configuration checked -arch x64&popd
..=..\$*
jitasm=set COMPlus_JitDisasm=$*
worksln="C:\Program Files (x86)\Microsoft Visual Studio\2019\Common7\IDE\devenv.exe" C:\git\work\somedir\work.sln
dbg=C:\debuggers\windbg.exe -o -y C:\git\work\bin\symboldirPDB -srcpath C:\git\work $*
...=..\..\$*
mydir=cd /d C:\this\is\mydir
```

To search for an alias or command, you could set below alias inside a macro file (see below for macro file)

```
c:\>doskey getkey=doskey /macros | findstr /i $*
c:\>getkey asm
jitasm=set COMPlus_JitDisasm=$*
```

<p/>

## Macro file

Instead of setting individual aliases using `doskey`, you could have all aliases in a text file and import it in at once using `/macrofile` switch.

```
C:\> more myaliases.txt
find=findstr /sin $*
bclr=pushd .&C:\runtime\build.cmd -subset clr -configuration checked -arch x64&popd
worksln="C:\Program Files (x86)\Microsoft Visual Studio\2019\Common7\IDE\devenv.exe" C:\git\work\somedir\work.sln
buildall=pushd C:\git\work&buildir\build.cmd x64 checked&popd 
dbg=C:\debuggers\windbg.exe -o -y C:\git\work\bin\symboldirPDB -srcpath C:\git\work $*
jitasm=set COMPlus_JitDisasm=$*
..=..\$*
...=..\..\$*
mydir=cd /d C:\this\is\mydir

C:\>doskey /macrofile=myaliases.txt

<< all aliases registered >>
```

If you want to update the aliases and re-import it, you could do something like this:

```
# Open the macro file imported
c:\>doskey okeys=notepad c:\myaliases.txt 

# Re-import macro file
c:\>doskey updatekeys=doskey /macrofile=C:\myaliases.txt

# Now you could update and re-import the aliases

c:\>okeys
<< update aliases.txt >>

c:\>updatekeys
```

<p/>

## Workflow

I usually have my aliases in a file `myaliases.txt`.  I wanted a way that every time I open my command prompt, all the aliases should be registered without me having to run the `doskey /macrofile` command manually. For that, I [created a DOS command prompt shortcut](https://www.makeuseof.com/tag/run-command-prompt-commands-desktop-shortcut/). Then, I opened the "Properties" of that shortcut and as shown below, in the "Target" section, execute the `doskey /macrofile` command.

  <img align="center" width="70%" height="70%" src="/assets/img/doskey/command-shortcut.JPG" />

Here is the complete command that I have in "Target" section:

```
C:\Windows\System32\cmd.exe /k "set reporoot=c:\git\runtime&&set BuildArch=arm64&&set BuildType=Debug&& doskey /MACROFILE=C:\myaliases.txt"
```

If you see above, I also set some environment variables like `reporoot` and `BuildArch`. I use these environment variables inside `myaliases.txt` and so I make sure that they are set before I import the `myaliases.txt` file as a `doskey macro`.

<p/>

## Conclusion

`doskey` proved to be a very productive command for me and I realized that there is no limit on how creative you could get with it. I hope you find some good use of this command to make your day-to-day DOS commands productive.

Namaste!