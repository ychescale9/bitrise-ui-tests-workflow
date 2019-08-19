# Bitrise UI Tests Workflow

Custom [bitrise.io](https://app.bitrise.io) workflow for running instrumented tests on emulators.

## Why do I need this?

Running proper automated UI tests on CI remains a challenge especially when operating with free plans for side projects. AFAIK **Bitrise** is the only service that supports running x86 emulators natively with `-gpu swiftshader`. They offer 1 free container for open-source projects. Other solutions I've looked at:

* CircleCI (and most other hosted CI providers) does not yet support running x86 emulators on their host machines which requires KVM support.
* Firebase Test Lab only allows 10 tests/day on virtual device and 5 tests/day on physical device with the Spark (free) Plan.
* Robolectric 4.x introduced support for running Espresso tests. While sharing test source between JVM and on-device tests mostly works, robolectric's shadowed environment is still too unstable / immature to be considered useful as they often have very different behaviors than when running on an emulator or real device.
* [Emulator 28.1.8 Canary](https://androidstudio.googleblog.com/2019/02/emulator-2818-canary.html) introduced a `emulator-headless` mode but with `-no-accel` on starting an emulator and installing APKs are still order of magnitude slower than running with `host` or `swiftshader` GPU acceleration.

Bitrise also provides the [Android Build for UI Testing](https://blog.bitrise.io/new-step-android-build-for-ui-testing) step whick takes care of spinning up an Emulator, waiting until it's fully booted, and runnning tests for the chosen module / build variant. But it has a couple of limitation:

* Only 1 build variant from 1 module can be run for each step. This means if you have multiple library modules with instrumented tests, you'll have to manually configure a **Android Build for UI Testing** step for each module (or setup parallel workflows for running these in parallel if you wish to pay for additional containers).
* Running tests in a library module doesn't work unless an app APK is also present. The [workaround](https://discuss.bitrise.io/t/vdt-not-able-to-run-instrumentation-tests-on-android-library-project/3197/7) is to also run `app:assembleDebug` for your library module tests.
* You don't have full control over the Emulator configs. E.g. the step doesn't use the lighter-weight `emulator-headless` which performs much better.

With a custom Workflow, we are able to control how we want to configure and run an Emulator on Bitrise with a custom shell script, and run all (or any) tests by running a single gradle command.

## Steps
1. Copy [start_emulator.sh](start_emulator.sh) into your `${projectRoot}/.bitrise/` (you could also just copy and paste it into a **Script** step in your Bitrise workflow directly).
2. Speficy `API_LEVEL` and `BUILD_TOOLS_VERSION` in the script. You could also use environment variables for these.
3. Add a **Script Runner** step to your Bitrise workflow and specify `.bitrise/start_emulator.sh` for **Script location**
4. Add a **Gradle Runner** step to your Bitrise workflow and specify `connectedDebugAndroidTest -Dorg.gradle.daemon=false -Dkotlin.incremental=false` for **Gradle task to run**. You could also use more fine-grained commands to only run tests for certain modules / build variants.
5. Push your change to trigger the build.

A sample `bitrise.yml` can be found [here](bitrise.yml).
