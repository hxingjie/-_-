# Config

## env:
    x86

## property:

### C/C++
```
General:
    Additional Include Directories: ...\GameLib\RealTime\include;%(AdditionalIncludeDirectories)
    Preprocessor: WIN32;_DEBUG;_WINDOWS;%(PreprocessorDefinitions)

All Options:
    debug: Runtime Library: Muti-thread Debug(/MTD)
    release: Runtime Library: Muti-thread Debug(/MT)
```

### Linker:
```
General:
    Additional Library Directories: ...\GameLib\RealTime\lib;%(AdditionalLibraryDirectories)
    Input:
        debug: GameLib_d.lib;%(AdditionalDependencies)
        release: GameLib.lib;%(AdditionalDependencies)
    System: Windows (/SUBSYSTEM:WINDOWS)
```


