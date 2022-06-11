# Marlin auto build

Marlin auto build is a build system for [Marlin](https://github.com/MarlinFirmware/Marlin) that allows you to build firmwares for your printer directly on github using [github actions](https://github.com/features/actions) without having to install anything on your local machine.  
The action can run on a schedule automatically and check for new stable releases plus the latest changes on the `bugfix-2.1.x` Marlin branch. Then it will apply your own configuration changes on them and build them.  
The system includes a little helper that allows you to define your changes using javascript. As Marlin evolves usually the default [configurations](https://github.com/MarlinFirmware/Configurations) change with it. Using the javascript configurator helps keep your own changes portable while new default configurations are released.

- [Marlin auto build](#marlin-auto-build)
  - [Getting started](#getting-started)
    - [Build files](#build-files)
  - [Configuration format](#configuration-format)
    - [Enabling an option](#enabling-an-option)
    - [Enabling an option with a value](#enabling-an-option-with-a-value)
    - [Enabling options with multiple values](#enabling-options-with-multiple-values)
    - [Disabling an option](#disabling-an-option)
    - [Raw text values with quote](#raw-text-values-with-quote)
  - [Extended builds and partials](#extended-builds-and-partials)
    - [Partials](#partials)
    - [Extra notes](#extra-notes)
  - [Build releases](#build-releases)
  - [Autogenerated Configurations](#autogenerated-configurations)
    - [Review the autogenerated configurations](#review-the-autogenerated-configurations)
    - [Creating builds based on other's configurations](#creating-builds-based-on-others-configurations)

## Getting started

If you are unfamiliar with github actions you can check the docs [here](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions) but all you have to do is the following:

1. Create a new empty repository.
2. In your repository, create the `.github/workflows/` directory to store your workflow files.
3. In the `.github/workflows/` directory, create a new file called `marlin_auto_build.yml` and add the following code.
```yml
name: marlin_auto_build

on:
  # trigger if we change a build
  push:
    paths:
      - 'builds/**'
  # trigger every night at 03:30
  schedule:
    - cron:  '30 3 * * *'

jobs:
  create_builds:
    runs-on: ubuntu-latest 
    steps:
    - uses: zisismaras/marlin_auto_build@v1

```
4. Create a `builds` directory in the root folder (this is where you will add your build files).
5. You can also add a `README.md` to describe your builds and a `LICENSE`. 
6. Commit these changes and push them to your GitHub repository.

You can take a look [here](https://github.com/zisismaras/ender_3_4.2.2_firmware) as a complete example.

### Build files

Build files should be saved in the `builds` directory as `.js` files (eg. `my-build.js`) and committed to git.  
Here is a build i am using for my Ender 3 that enables the bed leveling helper plus a little change required by octoprint.
```js
module.exports = {
    board_env: "STM32F103RE_creality",
    active: true,
    meta: {
        stable_name: "ender_3_4.2.2-{{marlin_version}}-{{uid}}",
        nightly_name: "ender_3_4.2.2-{{current_date}}-{{uid}}"
    },
    based_on: {
        repo: "https://github.com/MarlinFirmware/Configurations/",
        path: "/config/examples/Creality/Ender-3/CrealityV422/",
        stable_branch: "release-{{marlin_version}}",
        nightly_branch: "bugfix-2.1.x"
    },
    configuration: {
        enable: [
            //standard leveling menu helper
            "LCD_BED_TRAMMING",
            "BED_TRAMMING_INCLUDE_CENTER"
        ],
        disable: []
    },
    configuration_adv: {
        enable: [
            //octoprint
            "HOST_ACTION_COMMANDS"
        ],
        disable: []
    }
};
```
As you can see the build is basically a json file but writing it in javascript allows us to include comments and add any custom logic if needed.  
Let's explain all the different properties of the file.  

* `board_env`
  This is the env required by platformio to build the correct firmware for your specific motherboard. While we could infer this automatically (most of the time), having it hard-coded on the file is better since the system will run on its own without user interraction. Marlin has a little [guide](https://marlinfw.org/docs/basics/install_platformio_cli.html) on how to find the correct `board_env` for your printer.
* `active` Enables/disables the build. Default is `true`.
* `only` Set to `"stable"` or `"nightly"` to build only a single branch. By default both are built.
* `meta`
  Here we define the filenames we want for our 2 firmwares (stable and nightly). They can be whatever you want. There is also a set of variables (enclosed in `{{}}`) that you can use in them:
  - `{{marlin_version}}` The version of Marlin we are using (or the commit hash of the bugfix branch if it's a nightly build)
  - `{{current_date}}` The current date formatted as YYYYMMDDHHmm
  - `{{timestamp}}` The current date as a unix timestamp
  - `{{uid}}` A random 6-digit number
* `based_on`
  A pointer to a repository containing the base configuration we want to use. Usually we point at Marlin's default configurations but you can use any repo. The `path` should point to the directory that contains the actual configurations (`Configuration.h` and `Configuration_adv.h`).
* `configuration` and `configuration_adv` The options we want to enable or disable on the two files, more info below.

## Configuration format
All info below applies to both `configuration` and `configuration_adv`.  
Just make sure you are adding the change to the correct file!

### Enabling an option

To enable an option, add the option's name to the `enable` array:

```js
//...
enable: [
    "LCD_BED_TRAMMING",
    "BED_TRAMMING_INCLUDE_CENTER"
]
```

### Enabling an option with a value

If the option also has a value you can define it with a nested array like this:

```js
//...
enable: [
    ["GRID_MAX_POINTS_X", 5],
    ["CUSTOM_MACHINE_NAME", "My Printer"],
    ["INVERT_Z_DIR", false]
]
```
This will enable the options (if disabled) and update their values. If they are already enabled it will just update their value.

### Enabling options with multiple values

Some options require an array of values, you can define them like this:

```js
//...
enable: [
    ["NOZZLE_TO_PROBE_OFFSET", [0, 0, 0]]
    //output => #define NOZZLE_TO_PROBE_OFFSET { 0, 0, 0 } 
]
```

### Disabling an option

To disable an option, add the option's name to the `disable` array:

```js
//...
disable: [
    "SHOW_CUSTOM_BOOTSCREEN"
]
```

### Raw text values with quote

Some options need to reference other options or constants, for example:

```c
#define Z_PROBE_FEEDRATE_SLOW (Z_PROBE_FEEDRATE_FAST / 2)
#define GRID_MAX_POINTS_Y GRID_MAX_POINTS_X
```

We can use the `quote` (or `q` for short) tag to properly represent them:

```js
//...
enable: [
    ["Z_PROBE_FEEDRATE_SLOW", q`Z_PROBE_FEEDRATE_FAST / 2`],
    ["GRID_MAX_POINTS_Y", q`GRID_MAX_POINTS_X`]
]
```

Anything passed to `quote` will be written as is.  
For example the following option expects a char which we cannot represent in javascript:

```js
//...
enable: [
    ["AXIS4_NAME", q`'A'`]
    //output => #define AXIS4_NAME 'A'
]
```

## Extended builds and partials

For cases where you need to build multiple different firmwares for the same printer (or the same build for different printers), Marlin auto build includes an extension system to help reduce duplication and keep things nice and clean.    
Using the example build [above](#build-files) as a base (saved as `/builds/base.js`) we can define an extension build like this:

```js
module.exports = {
    extends: "builds/base.js", // <--
    meta: {
        stable_name: "ender_3_4.2.2-{{marlin_version}}-manual_mesh_5x5-{{uid}}",
        nightly_name: "ender_3_4.2.2-{{current_date}}-manual_mesh_5x5-{{uid}}"
    },
    configuration: {
        enable: [
            "PROBE_MANUALLY",
            ["NOZZLE_TO_PROBE_OFFSET", [0, 0, 0]],
            "MESH_BED_LEVELING",
            "RESTORE_LEVELING_AFTER_G28",
            "LCD_BED_LEVELING",
            "MESH_EDIT_MENU",
            ["GRID_MAX_POINTS_X", 5]
        ]
    }
};
```

This build includes all the changes on our base plus the required changes to enable the [Manual mesh leveling](https://marlinfw.org/docs/gcode/G029-mbl.html) feature of Marlin.

### Partials

Extended builds are great for simple scenarios like above but when you want to have more complex permutations of options things can get messy.  
Using the same example as above, let's move the manual mesh feature to a partial and then create 2 different builds.  
One will have a 5x5 grid which is pretty standard and the other one will have a 7x7 for that really messed up bed we have.

```js
// builds/manualBedMesh.js
module.exports = {
    partial: true, // <--
    configuration: {
        enable: [
            "PROBE_MANUALLY",
            ["NOZZLE_TO_PROBE_OFFSET", [0, 0, 0]],
            "MESH_BED_LEVELING",
            "RESTORE_LEVELING_AFTER_G28",
            "LCD_BED_LEVELING",
            "MESH_EDIT_MENU"
        ]
    }
};
```

```js
// builds/manualBedMesh5x5.js
module.exports = {
    extends: "builds/base.js", //we can extend and include at the same time
    include: "builds/manualBedMesh.js", // <--
    meta: {
        stable_name: "ender_3_4.2.2-{{marlin_version}}-manual_mesh_5x5-{{uid}}",
        nightly_name: "ender_3_4.2.2-{{current_date}}-manual_mesh_5x5-{{uid}}"
    },
    configuration: {
        enable: [
            ["GRID_MAX_POINTS_X", 5] // only the grid size is defined here
        ]
    }
};
```

```js
// builds/manualBedMesh7x7.js
module.exports = {
    extends: "builds/base.js", //we can extend and include at the same time
    include: "builds/manualBedMesh.js", // <--
    meta: {
        stable_name: "ender_3_4.2.2-{{marlin_version}}-manual_mesh_7x7-{{uid}}",
        nightly_name: "ender_3_4.2.2-{{current_date}}-manual_mesh_7x7-{{uid}}"
    },
    configuration: {
        enable: [
            ["GRID_MAX_POINTS_X", 7] // only the grid size is defined here
        ]
    }
};
```

### Extra notes

* Both `extends` and `includes` can accept an array of multiple builds (`includes: ["partial1.js", "partial2.js"]`). This is great for partials so we can have fine-grained control of the different set of changes we want to include in a build but for extended builds it is not recommended since you can end up with long inheritance chains which are confusing and hard to track.
* If you only want to use a base build for extensions but don't really need to use the base itself you can add `active: false` on it.
* If an option is defined in both the base and an extension, the extension takes precedence. You should check the action logs for warnings about conflicts.
* If an option is defined in both the base and a partial, the base takes precedence. You should check the action logs for warnings about conflicts.

## Build releases

After each action invocation any built firmwares will be available for download in the repository's github releases.  
Two releases will be created, one for stable builds and one for nightly builds.  
New releases are only created if there is a new stable Marlin version or a new commit on `bugfix`.  
Nightly releases are tagged as `prerelease` so the stable release is always shown as the latest.   
Updating a build will update the built file in the latest releases but deleting a build will not delete the already built files.

## Autogenerated Configurations

After each successful build, the autogenerated `Configuration.h` and `Configuration_adv.h` will be copied to the `autoGeneratedConfigs` folder
and committed in case you want to review them or if someone wants to base a new build upon them (see below).  
The directory structure is like this:  
* build name
  - stable
    * stable release tag (one for each built tag)
    * latest
  - nightly
    * commit hash (one for each built commit)
    * latest

### Review the autogenerated configurations

Next to each line that was changed there will be a comment with the original line so you can review the change.  
For example:  
```c
#define CUSTOM_MACHINE_NAME "Ender-3 4.2.2" //ORIGINAL: //#define CUSTOM_MACHINE_NAME "3D Printer"
```

### Creating builds based on other's configurations

Most of the time, you will base your builds on Marlin's example configurations but you might want to use
an existing modified build that someone else made and just add some more changes on top.  
For example:

```js
based_on: {
    repo: "https://github.com/some_user/some_repo",
    path: "/autogeneratedConfigs/the_build/{{releaseType}}/latest",
    stable_branch: "master",
    nightly_branch: "master"
}
```
The `{{releaseType}}` variable will be either `stable` or `nightly` depending on what's currently building.  
Instead of using `latest` you can point to the currently building marlin version by using `{{marlin_version}}` or hard-code a specific version that you want.  
**Note:** The other repo must have already built the version you are trying to build (so the configs are generated) or the process will fail. Using `latest` is recommended.  
