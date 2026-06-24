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

- Control-flow patterns
    - https://fsantanna.github.io/pingus/

# Porting process

**Never execute the game.**
Ask me for permission.

Between each step, provide feedback to the user:

1. designate a feature
2. execute locally
    - run offscreen (isolated from the real display)
        - `xvfb-run -a -s "-screen 0 800x600x24" pingus`
    - screenshot the virtual display
        - `import -window root out.png`
3. trace behavior
4. specify feature in a local plan
5. compare C++ and Céu versions
    - identify the control-flow patterns
6. update plan
7. propose in Atmos
8. implement
