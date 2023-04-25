<!--
SPDX-FileCopyrightText: Copyright 2022 EDF (Électricité de France S.A.)
SPDX-License-Identifier: BSD-3-Clause
-->
# CoAP/ACE PoC: Testing

## Automated tests

Several building blocks of the CoAP/ACE PoC contain automated test procedures at different levels.
Their success is generally tested for as part of their source code management at GitLab,
but can also be triggered manually.
All these tests indicate success by the processes exiting with a successful return value (0);
their output merely needs to be inspected in case of failure
(although it may show non-critical suggestions for improvement).

* The library crates ([ace-oscore-helpers], [coap-gatt-utils])
  run per module tests using Rust's test mechanisms built into cargo.
  `cargo test --all --all-features` can be used to run them manually.

* [coap-ace-poc-as] has a `test.py` file that starts up the server and exchanges requests with it.

* [liboscore] has several internal tests.
  Those presently relevant reside in `tests/native-rust`,
  and are executed using `cargo run`.

  This executes two series of tests:
  It runs the general libOSCORE test cases (written in C) against the Rust cryptography and message backends,
  and runs through the Rust wrappers of libOSCORE functionality in a Rust port of tests.

* [coap-ace-poc-firmware] and [coap-ace-poc-webapp]
  run through a part of the libOSCORE test suite at application startup.
  Failure to verify an internal test message prevents the application from reaching its operational state.

  This type of testing was chosen because it makes bugs visible quickly during development,
  because it ensures that the tests are run on the hardware and the browser where they are actually used,
  and because neither demo is performance sensitive enough that it would be detrimental to the user experience.

[coap-ace-poc-as]: https://gitlab.com/oscore/coap-ace-poc-as
[coap-ace-poc-firmware]: https://gitlab.com/oscore/coap-ace-poc-firmware
[coap-ace-poc-webapp]: https://gitlab.com/oscore/coap-ace-poc-webapp
[ace-oscore-helpers]: https://gitlab.com/oscore/ace-oscore-helpers
[coap-gatt-utils]: https://gitlab.com/oscore/coap-gatt-utils
[liboscore]: https://gitlab.com/oscore/liboscore

## Manual tests

### Main test

This section describes a large test run,
which covers all relevant parts of the combined proof-of-concept demo.
Its steps are combined into a single sequence to avoid tedious repetition.
Unless noted otherwise,
test steps are performed using the same client as the previous step.

The test is described to use two nRF52-DK devices
(one of which being attached to a debugger)
and two clients
(more precisely, a PC and an Android device, hereafter called "the phone").
This is to ensure that multiple connections work on all parts,
that the debugger is neither hindering nor impeding the device's operation,
and that different browser versions work.
The setup can be slimmed down to a single nRF52-DK on a single browser;
steps involving the second device are then simply skipped.
If no build environment for Rust is available,
both devices can be flashed using file upload;
steps involving debug output are then skipped.

Apart from steps skipped for lack of availability of devices,
the tests are designed to be executed in a single go from start to end.
While it is certainly possible to repeat single steps,
or to mix and match between devices and clients,
it is up to the user that all prerequisites for a step are met
when not operating in sequence.

The test is described with devices "d01" and "d02".
If the devices used were previously assigned different image numbers,
it is recommended to use those previous numbers again rather than picking 1 and 2.
(The reason for this recommendation is that operating systems may remember previously discovered devices' names by their MAC addresses,
resulting in inaccurate labels.)

#### Device setup

* Pick an nRF52-DK to act as "d01", and connect it to the PC.
  Download firmware image d01 from <https://oscore.gitlab.io/coap-ace-poc-firmware/>,
  and copy it onto the device following the instructions on <https://gitlab.com/oscore/coap-ace-poc-firmware/>.

  Two LEDs light up in a diagonal configuration.

* Open a Chrome based browser on the phone,
  and navigate to <https://oscore.gitlab.io/coap-ace-poc-webapp/>.

  Press "Add devices", and observe that a device "CoAP d01" is shown in the pairing dialog
  at latest after a few seconds.

  Cancel the operation by touching outside the dialog.

  (This check serves to verify that the device comes up right after flashing,
  and that cancelling discovery is harmless to the application).

* Disconnect d01 from the PC, and attach it to a plain power supply.

  (This serves to illustrate the independence of the device from the host,
  but also ensures that the devices do not get mixed up when flashing the second device).

* Pick a second nRF52-DK to act as "d02", and connect it to the PC.
  Turn it on, and flash its firmware through the debug interface
  by executing, inside the firmware's directory, the commands

  ```
  $ wget https://nsscprodmedia.blob.core.windows.net/prod/software-and-other-downloads/softdevices/s132/s132_nrf52_7.3.0.zip
  $ unzip s132_nrf52_7.3.0.zip s132_nrf52_7.3.0_softdevice.hex
  $ nrf-recover
  Do you want to continue [Y/n]: y
  $ cat uicr_reset_pin21.hex| grep -v '//' | probe-rs-cli download --chip nrf52832_xxAA --format hex /dev/stdin
  $ probe-rs-cli download --chip nrf52832_xxAA --format hex s132_nrf52_7.3.0_softdevice.hex
  $ RS_AS_ASSOCIATION=./configs/d02.yaml DEFMT_LOG=debug cargo +nightly run --release
  ```

  (Note that the first two commands can be skipped when this test is repeated;
  the other commands need to be executed every time the board is connected anew).

  After compilation and flashing, the device reports (among other things)

  ```
  INFO  Device is ready.
  INFO  OSCORE tests passed
  INFO  Advertising as connectable until a connection is establsihed
  ```

* Press "Add device" again on the phone.

  After a few seconds, it lists both the d01 and the d02 device in the pairing dialog.

  Abort the pairing.

#### PWA installation

* On the phone,
  enter the browser's menu,
  and press "Install app".

  The steps following that are dependent on the Android version and launcher used,
  but should result in the addition of a "CoAP/ACE PoC" icon on the home screen.

* Open the newly installed application.

  The phone displays a view similar to the web page previously shown
  (in particular, an "Add device" button),
  but no address bar.

  This application will be used for the remainder of the test.

#### Using pre-obtained credentials

*Starting from the following step, the next steps are time critical,
and need to be completed within 5 minutes.*

* On the phone, expand the "Add device manually" dropdown,
  and press "Add device manually" button without changing the default values.

  The devices list shows a "Device currently not connected",
  and a bold prompt to log in.

  Follow the login link. Pick "Log in as Junior".

  The application returns to the PoC webapp,
  again with an empty device list.

  Expand "Add device manually" again,
  and then the button.

  Again, an unconnected device is shown,
  this time with a token
  (but OSCORE not established).

* Disable WiFi and mobile data on the phone
  using its drop-down system menu.

  (Going through the full settings may reset the app
  on systems with very little RAM).

* Press the "Add device" button,
  and select d01.

  After a brief moment of showing two devices
  (when the webapp does not know yet that both its entries are the same),
  the devices list shows the "CoAP d01" device
  with a list of buttons,
  the same token as before,
  and "OSCORE: established".

* Press "Find".

  The device's LEDs spin briefly.

  Press "Read" in the "Temperature" line.

  The browser shows the device's temperature.

* Press "Idle LED state: dark".

  The command has no effect (because the user's role is "Junior").

*Time critical steps end here.*

* Press "Add device", and select d02.

  (Depending on the phone's Android version, there may be some delay).

  The device is listed with buttons,
  but Token and OSCORE show "none available" / "not established",
  and neither the Find nor the Read Temperature operation succeed.

#### Using a second device

* Direct a Chrome based browser on the PC to <https://oscore.gitlab.io/coap-ace-poc-webapp/>.

* Press "Add device" to add "d01".

  The devices list shows a "Device currently not connected",
  and a bold prompt to log in.

  Follow the login link. Pick "Log in as Senior".

* Press "Add device" to add "d01",
  and "Add device" again to add "d02".

  Both show in the device list,
  and have "OSCORE: established".

  For both devices,
  press "Find" and observe the respective device's LEDs spinning,
  "Read" and observe the temperature being shown in the browser,
  and the "bright" button turning all LEDs on until further notice.

*For the next step to pass the test,
at least 5 minutes need to have passed since the beginning of the above-mentioned time critical steps.*

* On the phone,
  press d01's "Find" button.

  There is no reaction from the device,
  and the browser now shows "Token: none available" below d01.

* Re-enable the phone's mobile and/or WiFi data communication,
  and wait for it to indicate connectivity.

* Press d01's "Find" button.

  The device's LEDs spin,
  and its entry in the device list now shows a token and established OSCORE.

* Repeat the above with d02.

* On the PC,
  set "d01" to "half",
  and observe now only two (diagonal) LEDs are active.

  (This demonstrates that different clients can be connected simultaneously).

#### Loss of state: Bluetooth

* On the terminal from which d02 was flashed,
  press Return several times.

  This creates a visual separation that can later be used to tell which messages were sent after this point in time.

* On the PC browser, press the "Disconnect" button of "d02".
  
  The debug console shows "Peer disconnected".

  The browser lists the device as "currently not conected",
  but still has a token and established OSCORE.

* Press "Add device" and select "d02".

  The debug console shows several lines of output,
  but only a single "Setting response" line that indicates a CoAP operation.

  (This is from where the browser updates the device's clock,
  which is not usually done this way in deployed systems).

  Press return a few more times in the terminal to create another separtion point.

* Press d02's "Find" button.

  The device's LEDs spin briefly.

  The debug console shows several lines of output,
  but only a single "Setting response" line that indicates a CoAP operation.

  This indicates that no setup messages need to be exchanged
  (the previous OSCORE context was reused).

#### Loss of state: Device power failure

* Remember the value of the token displayed for d01 in the PC browser.

* Turn the power switch of d01 to "off", and back to "on".

  The LEDs take their startup configuration (diagonal).

* Press "Find" on d01 in the PC's browser.

  The device entry switches to "not connected";
  the token value persists.

* Press "Add device", and add d01 again.

  The device's buttons show up again,
  OSCORE is established,
  and the token value has not changed.

  (This illustrates that the token can be used even after the Bluetooth device lost power).

#### Live token refresh

*For the next step to pass the test,
at least 5 minutes need to have passed since the phone was last used.*

* Remember the value of the token displayed for d02 on the phone.

* Press "Find" on d02.

  The LEDs do not spin,
  and the browser shows Token and OSCORE show "none available" / "not established".

  The debug output of d02 shows "Token has expired for (some number of) seconds",
  and that a very short response was sent ("Setting response [129]").

  Press Return on the debug console.

* Press "Find" on d02 again.

  The LEDs spin,
  a new Token value is displayed,
  and OSCORE is established again.

  The debug console shows (among other lines)
  two "Setting response" lines,
  indicating that a token exchange was followed
  by the actual (LED spinning) operation.

#### Housekeeping and clean-up

* On the PC, press the "Logout" button next to the single entry in the "Logins" section.

* Press d01's "Find" button.

  The LEDs spin.

  (This illustrates that the token's validity is unaffected by the browser's security association with the AS).

* Expand the "Add device manually" dropdown,
  and press the "Add device manually" button without changing any of the fields.

  (This has the side effect of attempting to obtain a new token.)

  For d01, Token and OSCORE show "none available" / "not established",
  and a "Login required" link is shown.

* Press d01's "Find" button.

  The LEDs do not spin.

This concludes the main test series.
Devices should be switched off after the conclusion of the test
to avoid draining their batteries.

### Extra tests

These small tests are tedious to set up,
but cover relevant corner cases that are otherwise not testable.

* **BLE connection saturation**:
  In the firmware's `src/main.rs`,
  reduce `MAX_CONNECTIONS` and the `pool_size` above `fn blueworker` to one less than the number of available devices.
  Flash the modified firmware with debug output at "info" level (as in the main test),
  and run the web app from all the available devices.

  When all but the last device are connected,
  the device's debug output regularly prints "Connections full; advertising unconnectable",
  but these devices are usable.
  When the last device attempts a connection,
  it fails to establish the connection,
  but the previous devices keep working.

* **WASM panics**:
  In the webapp's `src/main.rs`,
  add a line `panic!();` inside the case for `ScanBLE` (in `impl Component for Model` / `update`).
  Rebuild as per the webapp's documentation,
  access it from the browser,
  and press the "Add device" button.

  The application shows a "This application has crashed, this is a bug. [...]" message in a red box,
  and otherwise becomes unresponsive until it is reloaded.

----

This project and all files contained in it is published under the
BSD-3-Clause license as defined in [`LICENSES/BSD-3-Clause.txt`](LICENSES/BSD-3-Clause.txt).

Copyright: 2022 EDF (Électricité de France S.A.)

Author: Christian Amsüss
