[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=S2-group_android-runner&metric=alert_status)](https://sonarcloud.io/dashboard?id=S2-group_android-runner)
[![Build Status](https://travis-ci.org/S2-group/android-runner.svg?branch=master)](https://travis-ci.org/S2-group/android-runner)
[![Coverage Status](https://coveralls.io/repos/github/S2-group/android-runner/badge.svg?branch=master)](https://coveralls.io/github/S2-group/android-runner?branch=master&service=github)
# Android Runner
Automated experiment execution on Android devices

## Install
This tool is only tested on Ubuntu, but it should work in other linux distributions.
You'll need:
- Python 3
- Android Debug Bridge (`sudo apt install android-tools-adb`)
- Android SDK Tools (`sudo apt install monkeyrunner`)
- JDK 8 (NOT JDK 9) (`sudo apt install openjdk-8-jre`)
- lxml (`sudo apt install python-lxml`)
- Install Python requirements (`pip install -r requirements.txt`)

Additionally, the following are also required for the Batterystats method:
- power_profile.xml (retrievable from the device using [APKTool](https://github.com/iBotPeaches/Apktool))
- systrace.py (from the Android SDK Tools)
- A device that is able to report on the `idle` and `frequency` states of the CPU using systrace.py

Note: It is important that monkeyrunner shares the same adb the experiment is using. Otherwise, there will be an adb restart and output may be tainted by the notification.

Note 2: You can specifiy the path to adb and/or Monkeyrunner in the experiment configuration. See the Experiment Configuration section below.

Note 3: To check whether the the device is able to report on the `idle` and `frequency` states of the CPU, you can run the command `python systrace.py -l` and ensure both categories are listed among the supported categories.

## Quick start
To run an experiment, run:
```bash
python android_runner your_config.json
```
Example configuration files can be found in the subdirectories of the `example` directory.

## Structure
### devices.json
A JSON config that maps devices names to their adb ids for easy reference in config files.

### Experiment Configuration
Below is a reference to the fields for the experiment configuration. It is not always updated.

**adb_path** *string*
Path to adb. Example path: `/opt/platform-tools/adb`

**monkeyrunner_path** *string*
Path to Monkeyrunner. Example path: `/opt/platform-tools/bin/monkeyrunner`

**systrace_path** *string*
Path to Systrace.py. Example path: `/home/user/Android/Sdk/platform-tools/systrace/systrace.py`

**powerprofile_path** *string*
Path to power_profile.xml. Example path: `android-runner/example/batterystats/power_profile.xml`

**type** *string*
Type of the experiment. Can be `web`, `native` or 'plugintest'

**device_spec** *string*
Specify this property inside of your config to specify a `devices.json` outside of the Android Runner repository. For example:

 ```js
 {
   // ....
   "type": "native",
   "devices_spec": "/home/user/experiments/devices.json",
   "devices": {
     "nexus6p": {}
   },
   // ...
 }
 ```

**repetitions** *positive integer*
Number of times each experiment is run.

**clear_cache** *boolean*
Clears the cache before every run for both web and native experiments.  Default is *false*.

**randomization** *boolean*
Random order of run execution. Default is *false*.

**duration** *positive integer*
The duration of each run in milliseconds, default is 0. Setting a too short duration may lead to missing results when running native experiments, it is advised to set a higher duration time if unexpected results appear.

**reset_adb_among_runs** *boolean*
Restarts the adb connection after each run.  Default is *false*.  Recommended to run Android Runner as a privileged user to avoid potential issues with adb device authorizaton.

**time_between_run** *positive integer*
The time that the framework waits between 2 successive experiment runs. Default is 0.

**devices** *JSON*
A JSON object to describe the devices to be used and their arguments. Below are several examples:
```js
  "devices": {
    "nexus6p": {
      "root_disable_charging": "True",
      "charging_disabled_value": 0,
      "usb_charging_disabled_file": "/sys/class/power_supply/usb/device/charge",
      "device_settings_reqs": {"e.www.gyroscopetest": ["location_high_accuracy", ...], ...}
      }
    }
  }
```

```js
  "devices": {
    "nexus6p": {
      "root_disable_charging": "False"
    }
  }
```

```js
  "devices": {
    "nexus6p": {}
  }
```
Note that the last two examples result in the same behaviour.

The **root_disable_charging** option specifies if the devices needs to be root charging disabled by writing the **charging_disabled_value** to the **usb_charging_disabled_file**. Different devices have different values for the **charging_disabled_value** and **usb_charging_disabled_file**, so be careful when using this feature. Also keep an eye out on the battery percentage when using this feature. If the battery dies when the charging is root disabled, it becomes impossible to charge the device via USB.

**device_settings_reqs** can be set to programmatically enable and disable settings on the test device.  It was added to automate the process of turning on and off location services in a randomized experiment where some applications required it and others that didn't.  Two options available currently: location services with the help of Google and one without.  **location_high_accuracy** is the option for location services with Google; **location_gps_only** is the other.  More adb commands are likely to be added in the future that work for other sensors.  Turn off location services before the experiment starts.

**WARNING:** Always check the battery settings of the device for the charging status of the device after using root disable charging.
If the device isn't charging after the experiment is finished, reset the charging file yourself via adb su command line using:
```shell
adb su -c 'echo <charging enabled value> > <usb_charging_disabled_file>'
```

**paths** *Array\<String\>*
The paths to the APKs/URLs to test with. In case of the APKs, this is the path on the local file system.

**apps** *Array\<String\>*
The package names of the apps to test when the apps are already installed on the device. For example:
```js
  "apps": [
    "org.mozilla.firefox",
    "com.quicinc.trepn"
  ]
```

**browsers** *Array\<String\>*
*Dependent on type = web*
The names of browser(s) to use. Currently supported values are `chrome`.

**profilers** *JSON*
A JSON object to describe the profilers to be used and their arguments. Below are several examples:
```js
  "profilers": {
    "trepn": {
      "sample_interval": 100,
      "data_points": ["battery_power", "mem_usage"]
    }
  }
```

```js
  "profilers": {
    "android": {
      "sample_interval": 100,
      "data_points": ["cpu", "mem"],
      "subject_aggregation": "user_subject_aggregation.py",
      "experiment_aggregation": "user_experiment_aggregation.py"
    }
  }
```

```js
  "profilers": {
    "batterystats": {
      "cleanup": true,
      "subject_aggregation": "default",
      "experiment_aggregation": "default",
      "enable_systrace_parsing": true,
      "python2_path": "python2"
    }
  }
```

```json
  "profilers": {
    "Garbagecollection": {
      "subject_aggregation" : "default"
    }
  }
```
The garbage collection (GC) plugin gathers and counts GC log statements by searching in adb's logcat for logs that meet the format of a GC call as described [here](https://dzone.com/articles/understanding-android-gc-logs).
The default subject aggregation lists the counted GC calls in a single file for easy further processing.
```json
  "profilers": {
    "Frametimes": {
      "subject_aggregation" : "default",
      "sample_interval": 1000
    }
  }
```
The frame times plugin gathers unique frame rendering durations (in nanoseconds) by utilizing `dumpsys gfxinfo framestats` and counts the amount of delayed frames that occurred following the 16ms threshold [defined by Google](https://developer.android.com/training/testing/performance).
The sample interval is configurable but advised to keep under 120 seconds as the framestats command returns only data from frames rendered in the past 120 seconds as described [here](https://developer.android.com/training/testing/performance).
Shorter sample intervals will not cause duplication in the frames gathered as only unique frames are kept.
The default subject aggregation consists of combining both the frametimes as the delayed frames count in single files for easy further processing.

**subject_aggregation** *string*
Specify which subject aggregation to use. The default is the subject aggregation provided by the profiler. If a user specified aggregation script is used then the script should contain a ```bash main(dummy, data_dir)``` method, as this method is used as the entry point to the script.

**experiment_aggregation** *string*
Specify which experiment aggregation to use. The default is the experiment aggregation provided by the profiler. If a user specified aggregation script is used then the script should contain a ```bash main(dummy, data_dir, result_file)``` method, as this method is used as the entry point to the script.

**cleanup** *boolean*
Delete log files required by Batterystats after completion of the experiment. The default is *true*.

**enable_systrace_parsing** *boolean*
The Batterystats profiler uses the profiling tool Systrace internally to measure CPU specific activity and energy consumption on the mobile device. For some devices the parsing of the output of Systrace fails, causing the experiment run to fail. You can safely disable the Systrace parsing when you encounter Systrace parsing errors given that your experiment does not need rely on CPU specific information, but rather on the overall energy consumption of the mobile device. The overall energy consumption is not affected by the Systrace logs since it is tracked using another tool. The default is *true*.

**python2_path** *string*
The path to python 2 that is used to launch Systrace. The default is *python2*.

**scripts** *JSON*
A JSON list of types and paths of scripts to run. Below is an example:
```js
"scripts": {
  "before_experiment": "before_experiment.py",
  "before_run": "before_run.py",
  "interaction": "interaction.py",
  "after_run": "after_run.py",
  "after_experiment": "after_experiment.py"
}
```
Below are the supported types:
- before_experiment
  executes once before the first run
- before_run
  executes before every run
- after_launch
  executes after the target app/website is launched, but before profiling starts
- interaction
  executes between the start and end of a run
- before_close
  executes before the target app/website is closed
- after_run
  executes after a run completes
- after_experiment
  executes once after the last run

Instead of a path to string it is also possible to provide a JSON object in the following form:
```js
    "interaction": [
      {
        "type": "python2",
        "path": "Scripts/interaction.py",
        "timeout": 500,
        "logcat_regex": "<expr>"
      }
   ]
```
Within the JSON object you can use "type" to "python2", "monkeyrunner" or, "monkeyreplay" depending on the type of script. "python2" can be used for a standard python script,  "monkeyreplay" for running a Monkeyrunner script with the use of the Monkeyrunner framework and "monkeyrunner" can be used to run a Monkeyrunner directly without the entire Monkeyrunner framework. The "timeout" option is to set a maximum run time in miliseconds for the specified script. The optional option "logcat_regex" filters the logcat messages such that it only keeps lines where the log message matches "\<expr\>" where "\<expr\>" is a regular expression.

For the option `monkeyreplay`, the name of the file is used to differentiate the replays executed for specific apps and replays to be executed for all apps. 
To specify a replay file for all runs, name the replay file `monkey_replay_all`.
To specify a replay file for a specific app, name the replay file using the app package. 
The replay file is executed if the replay file name is contained within the app package. 
To allow using the same replay file for the same app with different treatments, a partial match is performed instead of a full match.
For example, for the apps wih package `my.app1.treatement1` and `my.app1.treatmenet2` or `treatement1.my.app1` and `treatmenet2.my.app1`, name the replay file `my.app1` and it will be played for both treatments, but not for the app with package `my.app2.treatement1`.

### Monkey Recorder

First of all, be aware that the recorder is old and not 100% reliable.
To use the Monkey Recorder to obtain interactions with an app, first, connect the mobile device where the experiment will run and execute the command:

```bash
monkeyrunner MonkeyPlayer/monkey_recorder.py
```

This will open a window with a mirror from the connected device, as illustrated in the figure below.

<p align="center">
	<img src="docs/img/MonkeyRecorder.png" alt="Monkey Recorded app" width="75%"/>
</p>

You might have the error
```bash
Exception in thread "main" java.awt.AWTError: Assistive Technology not found: org.GNOME.Accessibility.AtkWrapper
```

You can resolve it by editing the file `/etc/java-8-openjdk/accessibility.properties` and commeting the line

```
assistive_technologies=org.GNOME.Accessibility.AtkWrapper
```



This window will register your interactions in the right panel, which can be exported to a file by clicking at `Export Actions`.
Important aspects:

* It will only register the interactions made within this window. It does not capture the interactions made directly in the device.
* The mirror window is slow, really slow, so perform only one action at a time.
* The exported actions must be translated to a format recognized by AR:

```text
# Exported format
TOUCH|{'x':279,'y':1045,'type':'downAndUp',}
TOUCH|{'x':747,'y':1008,'type':'downAndUp',}
TOUCH|{'x':94,'y':176,'type':'downAndUp',}

# Read format
{"type": "touch", "y": 1045, "down": 0, "up": 1, "x": 279 }
{"type": "touch", "y": 1008, "down": 2, "up": 3, "x": 747 }
{"type": "touch", "y": 176, "down": 3, "up": 4, "x": 94 }
```

Be sure to have only one interaction per line and don't break interactions in multiple lines.
If your interaction needs to wait for resources to load (e.g., requesting an HTTP request, navigating to an activity, etc.), a wait statement must be added after the interaction. 
You can do this by adding a `sleep` property in the interaction with the time in seconds to wait.

```
{"type": "touch", "y": 1045, "down": 0, "up": 1, "x": 279, "sleep": 4 }
```

To verify the possible interactions check this [example file](examples/native/Scripts/monkey.txt) and the [Replay](MonkeyPlayer/replayLogic.py) script that parse and run the replay file


## Plugin profilers
It is possible to write your own profiler and use this with Android runner. To do so write your profiler in such a way
that it uses [this profiler.py class](AndroidRunner/Plugins/Profiler.py) as parent class. The device object that is mentioned within the profiler.py class is based on the device.py of this repo. To see what can be done with this object, see the source code [here](AndroidRunner/Device.py).

You can use your own profiler in the same way as the default profilers, you just need to make sure that:
- The profiler name is the same as your python file and class name.
- Your python file isn't called 'Profiler.py' as this file will be overwritten.
- The python file is placed in a directory called 'Plugin' which resided in the same directory as your config.json

To test your own profiler, you can make use of the 'plugintest' experiment type which can be seen [here](examples/plugintest/)

## Experiment continuation
In case of an error or a user abort during experiment execution, it is possible to continue the experiment if desired. This is possible by using a ```--progress``` tag with the starting command. For example:

```python android_runner your_config.json --progress path/to/progress.xml```

## Detailed documentation
The original thesis can be found here:
https://drive.google.com/file/d/0B7Fel9yGl5-xc2lEWmNVYkU5d2c/view?usp=sharing

The thesis regarding the implementation of Batterystats can be found here:
https://drive.google.com/file/d/1O7BqmkRFRDq7AD1oKOGjHqJzCTEe8AMz/view?usp=sharing

## FAQ
### Devices have no permissions (udev requires plugdev group membership)
This happens when the user calling adb is not in the plugdev group.
#### Fix
`sudo usermod -aG plugdev $LOGNAME`
#### References
https://developer.android.com/studio/run/device.html

http://www.janosgyerik.com/adding-udev-rules-for-usb-debugging-android-devices/

### [Batterystats] IOError: Unable to get atrace data. Did you forget adb root?
This happens when the device is unable to retrieve CPU information using systrace.py.
#### Fix
Check whether the device is able to report on both categories `freq` and `idle` using Systrace:

`python systrace.py -l`

If the categories are not listed, use a different device.
#### References
https://developer.android.com/studio/command-line/systrace
