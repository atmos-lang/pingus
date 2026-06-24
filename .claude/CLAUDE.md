# Project

Port of Pingus to the programming language Atmos.

- Skills
    - `atmos.md`
    - `pico-lua.md`
    - `pico-atmos.md`

- Original code in C++
    - `./pingus.cpp/`
    - https://github.com/Pingus/pingus
    - run locally (from ubuntu package)
        - `pingus`

- Old port to Céu
    - `./pingus.ceu/`
    - https://github.com/fsantanna/pingus
    - https://fsantanna.github.io/pingus/

# Porting process

1. designate a feature
2. execute locally
    - run offscreen (isolated from the real display)
        - `xvfb-run -a -s "-screen 0 800x600x24" pingus`
    - screenshot the virtual display
        - `import -window root out.png`
3. trace behavior
3. specify feature in a local plan
4. compare C++ and Céu versions
5. propose in Atmos
6. update plan
7. implement
