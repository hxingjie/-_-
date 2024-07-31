#

x86

property: C/C++
  general:
    Additional Include Directories: ...\GameLib\2DGraphics1\include;%(AdditionalIncludeDirectories)
    Preprocessor: WIN32;_DEBUG;_WINDOWS;%(PreprocessorDefinitions)

  Linker:
    general:
      Additional Library Directories: ...\GameLib\2DGraphics1\lib;%(AdditionalLibraryDirectories)
    Input: GameLib_d.lib;%(AdditionalDependencies)
    System: Windows (/SUBSYSTEM:WINDOWS)
