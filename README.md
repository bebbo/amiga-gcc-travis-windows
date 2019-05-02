# crap0000

Experiments to get amiga-gcc building for Windows with travis-ci

## windows builds are creeping slow

The current approach splits the build in several jobs. 

 - The assembled result is transported via cache.

 
```
os: windows
language: shell
cache:
  directories:
  - /c/Users/travis/build/bebbo/crap0000/amiga/
```

 - The first stage initializes the msys64 build system and cleares the previous build result:
 ```
   - stage: Setup cache
    script:
    - rm -rf amiga
    - mkdir -p amiga
 ```
The next stage builds the first targets
 - Note that I'm starting msys64 via powershell (seems to work best).
 - Also make is invoked as background task and the foreground task does waiting and writing some output to avoid the 10 minute-no-output limit!
 - Note that `kill -0 $pid` did not work for me, so it's wating for the last file created by make!!
 - Another trick is to disable Windows Defender - gains some speed too!!!
```
- stage: make binutils
    before_install:
    - powershell -command 'Set-MpPreference -DisableRealtimeMonitoring $true'
    - if [ ! -f msys64/usr/bin/bash.exe ]; then wget --progress=bar:force https://franke.ms/msys64.tgz; rm -rf msys64; tar xzf msys64.tgz; fi
    script:
    - git clone https://github.com/bebbo/amiga-gcc
    - powershell -executionpolicy bypass -command "msys64\\usr\\bin\\bash.exe --login -c 'cd amiga-gcc && (PREFIX=$(realpath amiga) make -j3 info binutils gdb gprof&); while [ ! -f $(realpath amiga)/bin/m68k-amigaos-gdb ]; do date; sleep 30; done'"
```

- The following stages are using another trick
Since the build results are missing, all files to be checked are faked. Thus the previous build stuff isn't built again.
```
 - stage: make ndk
    before_install:
    - powershell -command 'Set-MpPreference -DisableRealtimeMonitoring $true'
    - if [ ! -f msys64/usr/bin/bash.exe ]; then wget --progress=bar:force https://franke.ms/msys64.tgz; rm -rf msys64; tar xzf msys64.tgz; fi
    script:
    - git clone https://github.com/bebbo/amiga-gcc
    - mkdir -p amiga-gcc/projects/binutils
    - touch    amiga-gcc/projects/binutils/configure
    - mkdir -p amiga-gcc/build-MSYS_NT-10.0/binutils
    - touch    amiga-gcc/build-MSYS_NT-10.0/binutils/Makefile
    - touch    amiga-gcc/build-MSYS_NT-10.0/binutils/_done
    - powershell -executionpolicy bypass -command "msys64\\usr\\bin\\bash.exe --login -c 'cd amiga-gcc && PREFIX=$(realpath amiga) make -j3 info ndk ndk13'"
```
- Rinse and repeat until all is done.

- The final step uses the assembled stuff and creates a setup executable

# Done
