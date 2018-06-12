# nRF52832-buttonless-dfu-development-tutorial

This Windows tutorial is valid as of 2018-06-06, updates to tools and SDKs might have been made since then. Please review the information on our Infocenter.nordicsemi.com to see if any updates have been made.

This tutorial does not cover the installation and use of all the Nordic tools that are required to do follow this tutorial, the assumption is that you already know [some of the basics](http://infocenter.nordicsemi.com/topic/com.nordic.infocenter.gs/dita/gs/gs.html?cp=1) for developing with a nRF SOC. Links are provided where I thought they would be needed, so please review this, and also use our [DevZone](http://devzone.nordicsemi.com/) portal to find answers to questions you might have.


## Requirements
- SDK v15.0.0
- [SEGGER Embedded Studio](https://www.youtube.com/user/NordicSemi/) - see youtube playlist for information on setup
- armgcc compiler and make
- git
- nrfutil version 3.5.1, any later version should be fine to use, but might require some modifications to the scripts 
- nrfjprog and mergehex version 9.7.1 or later
- 2x nRF52-DK (nRF52832 development kit, PCA10040)

To follow this tutorial, and if you prefer to not edit any of the provided .bat files, please do the following:
1. Download SDK v15.0.0 from here: http://developer.nordicsemi.com/nRF5_SDK/nRF5_SDK_v15.x.x/, and extract it into “C:\” drive, matching the path: “C:\nRF5_SDK_15.0.0_a53641a\license.txt”. I also suggest you make the entire folder into a .git repository as that makes it easier to see which changes that have been made later on.
2. Make sure [nRF5x Command Line Tools]( http://infocenter.nordicsemi.com/topic/com.nordic.infocenter.tools/dita/tools/nrf5x_command_line_tools/nrf5x_command_line_tools_lpage.html?cp=5_1), [nRF5x CLT download](http://www.nordicsemi.com/eng/Products/Bluetooth-low-energy/nRF51822/nRF5x-Command-Line-Tools-Win32/(language)/eng-GB) is installed
3. Install [nrfutil](http://infocenter.nordicsemi.com/topic/com.nordic.infocenter.tools/dita/tools/nrfutil/nrfutil_installing.html?cp=5_5_1) using Python pip.
4. Make sure [nRF Connect](http://infocenter.nordicsemi.com/topic/com.nordic.infocenter.tools/dita/tools/nRF_Connect/nRF_Connect_framework_intro.html?cp=5_3) is installed and that you know how to use it. The reason we require 2x nRF52-DKs, is that nRF Connect requires its own DK in order to run and perform the DFU updates. Please run through the nRF Connect Bluetooth low energy guide, and make sure you are able to establish a connection to a BLE device from the PC.
5. Make sure [SEGGER Embedded Studio](http://infocenter.nordicsemi.com/topic/com.nordic.infocenter.gs/dita/gs/nordic_tools.html?cp=1_1) is installed and you know how to use it. 

You can always edit the paths in the .bat files if you prefer to extract the SDK into some other folder on your machine.


## Step 1 - Test Buttonless DFU Template Application
Go through the description here: http://infocenter.nordicsemi.com/topic/com.nordic.infocenter.sdk5.v15.0.0/ble_sdk_app_buttonless_dfu.html?cp=4_0_0_4_1_2_6, and make sure that you understand how this works. I will roughly list the steps below for the approach without bonds.

The steps you are going to do as part of testing Buttonless Secure DFU without bonds:
1.	Flash the precompiled hex file using nrfjprog to the device DK (not the DK we are going to use with nRF Connect)
```
nrfjprog --program C:\nRF5_SDK_15.0.0_a53641a\examples\dfu\secure_dfu_test_images\ble\nrf52832\sd_s132_bootloader_buttonless_with_setting_page_dfu_secure_ble_debug_without_bonds.hex --chiperase
```
2.	Power off the device DK
3.	Connect the other DK that has not been programmed to the PC, open up nRF Connect BLE and use this other DK to start scanning for devices (choose serial port, program FW if not done already, start scan)
4. Power on the device DK that was programmed with the DFU test image, it should advertise as `Nordic_Buttonless`
5. Connect to the device DK from nRF Connect
6. Click on the DFU icon that should have appread in nRF Connect, and select the `hrs_application_s132.zip` from the same folder as the DFU test image FW. It should list package info which includes an application. Click Start DFU
7. Complete the update, then see that the HRS application starts up by scanning for it with nRF Connect. The name will now be `Nordic_HRM`, and if you connect, you can see the heart rate and battery updates as with the standard HRM example

This is what we are going to try and mimic when we are “developing” our own buttonless DFU FW. I say “developing” because we are not actually going to do any changes or development besides re-compiling the provided source files.



## Step 2 - Create our own bootloader using our own private/public keys
Read through the instructions as provided for the BLE Secure DFU Bootloader on the infocenter: http://infocenter.nordicsemi.com/topic/com.nordic.infocenter.sdk5.v15.0.0/ble_sdk_app_dfu_bootloader.html?cp=4_0_0_4_1_3. A script is provided with this tutorial and this .bat can be used to create the private/public key pair using nrfutil, 01_generate_private_public_keys.bat. See http://infocenter.nordicsemi.com/topic/com.nordic.infocenter.tools/dita/tools/nrfutil/nrfutil_intro.html?cp=5_5 for more information on nrfutil.

1.	Create a private and public key using nrfutil by running script `01*.bat`, this will generate `private.pem` and `dfu_public_key.c`
2.	Copy the generated dfu_public_key.c into `C:\nRF5_SDK_15.0.0_a53641a\examples\dfu` folder, replacing the dfu_public_key file that is already in there.
3.	Install micro-ecc. See infocenter [BLE Secure DFU Bootloader](http://infocenter.nordicsemi.com/topic/com.nordic.infocenter.sdk5.v15.0.0/ble_sdk_app_dfu_bootloader.html?cp=4_0_0_4_3_0), and [Installing micro-ecc](http://infocenter.nordicsemi.com/topic/com.nordic.infocenter.sdk5.v15.0.0/lib_crypto_backend_micro_ecc.html?cp=4_0_0_3_11_16_2_2#lib_crypto_backend_micro_ecc_install). When all the tools are installed (armgcc, make, git), you can run the `C:\nRF5_SDK_15.0.0_a53641a\external\micro-ecc\build_all.bat` script to install it
4.	Compile `C:\nRF5_SDK_15.0.0_a53641a\examples\dfu\secure_bootloader\pca10040_ble\ses\secure_bootloader_ble_s132_pca10040.emProject`, which is the secure BLE bootloader for nRF52832
5.	Copy the output bootloader .hex from `\Output\Release\Exe\secure_bootloader_ble_s132_pca10040.hex` into the same folder as the tutorial scripts, then rename it `bootloader.hex`
6.	Program the `s132_nrf52_6.0.0_softdevice.hex`, as well as the `bootloader.hex` to the nRF52832 by running the script `02*.bat`. If you have 2 DKs connected to the PC when you run this script, it will prompt you to select the Serial number of the DK you want to program. Select the Serial number of the device DK that is not used for nRF Connect
7.	Power cycle the device DK, and it should start up and enter DFU mode and advertise as `DfuTarg`
8. Restart scanning from nRF Connect, and you should see this advertisement

At this point, you know that you can create your own bootloader with your own keys, and it will make the DK run and enter DFU mode.



## Step 3 - Create our own FW.zip package which we can upload to the device over BLE
To create our own firmware package, we will be using `nrfutil` as we did in step 1. Please review the following page for more information: http://infocenter.nordicsemi.com/topic/com.nordic.infocenter.sdk5.v15.0.0/lib_bootloader_dfu_validation.html?cp=4_0_0_3_5_1_1_1#lib_dfu_image. We will be using a script to generate the .zip package `03_create_fw_zip_package.bat`. This script assumes there is an `app.hex`, as well as the `s132_nrf52_6.0.0_softdevice.hex` present in the same folder as the script. There is also a `03b_create_fw_zip_package_noSD.bat` that can be used to generate a `FW.zip` without a SD.

### Generating firmware .zip package with application and SoftDevice
1.  Make sure the s132 softdevice .hex file `s132_nrf52_6.0.0_softdevice.hex` is in the same folder as the script
2.  Copy any application .hex file into the same folder as the script, and rename this file `app.hex`. I will be using the HRS application from `C:\nRF5_SDK_15.0.0_a53641a\examples\ble_peripheral\ble_app_hrs\pca10040\s132\ses\Output\Release\Exe\ble_app_hrs_pca10040_s132.hex`; you will have to compile this example to get the .hex file. Other examples can be used as well. **Note: You cannot use the precompiled .hex files provided with the examples as these include the SoftDevice as well as the application FW**
3.  Run the `03_create_fw_zip_package.bat` to generate a `FW.zip` package with SD + APP that can be uploaded to the DK over BLE DFU
    1.  We will be using application-version 1 just because for demo purposes.
    2.  We will be using application-version-string “1.0.0” for demo purposes.
    3.  We will not be uploading a new bootloader to the device
    4.  We will be requiring hw-version 52
    5.  We will be requiring sd-req 0xA8 as this matches s132 v6.
    6.  We will be using sd-id 0xA8 as we are uploading the same SD.
    7.  We will be using the private key `private.pem` we generated earlier as input to --key-file
4.  Once the `FW.zip` is generated, power cycle the device DK and make sure it starts advertising in bootloader mode as DfuTarg. If it does not, go back to the previous step and make sure that works
5.  Connect to DfuTarg from nRF Connect
6.  Click the DFU button, and browse to the `FW.zip` file we just created and select this one, click “Start DFU”. The SD will be uploaded first, then the application
7.  The DFU process should finish without any errors, and once it’s done, the DK should now be advertising the application image you just uploaded, which in my case is the HRS example. You should verify this using nRF Connect – scan and find the ADV name `Nordic_HRM`

At this point, you should now have a working process for uploading new SD and APP images to the DK using BLE DFU and the private/public key pair you used earlier and which matches your bootloader.


### Generating firmware .zip package with application only
If you would like to test another FW image besides the first one we just uploaded, and maybe without including the SoftDevice, do the following;

1. Copy another application .hex file into the folder and rename this app.hex, then run `03b_create_fw_zip_package_noSD.bat` to create a new `FW.zip` package, this time only including an application
    1. I used my favorite application ble_app_uart, copied in the compiled .hex file from `C:\nRF5_SDK_15.0.0_a53641a\examples\ble_peripheral\ble_app_uart\pca10040\s132\ses\Output\Release\Exe\ble_app_uart_pca10040_s132.hex`, renamed it app.hex, ran the `03b_*.bat` script to create the `FW.zip`
2.  To get the DK back into DFU mode, hold button 4 while powering on the device DK
    1. It should enter DFU mode and `DfuTarg` should show up in nRF Connect when scanning
3.  Connect to `DfuTarg`, then follow the same procedure as above to upload the new `FW.zip` package – you will now see that there is only app data in the .zip package, no SD included
4.  The DFU process should finsh without any errors, and the DK is now advertising as `Nordic_UART`





