## Connecting IoT devices to the cloud with IBM Watson IoT and mbed Cloud Connect

### Introduction

This tutorial explains how to connect your mbed device to Watson IoT, using mbed Cloud Connect and Watson’s Connector Bridge.

By the end, your mbed device will be able to deliver data to the Watson IoT analytics service.

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/overview.png)</span>

It guides you through the following tasks:

1. [Log into the online IDE and import the K64F project](#log-into-the-online-ide-and-import-the-k64f-project)
1. [Create your mbed Cloud Connect Access Key](#create-your-mbed-connector-access-key)
1. [Set the developer credentials in your endpoint code](#set-the-developer-credentials-in-your-endpoint-code)
1. [Run your endpoint code](#run-your-endpoint-code)
1. [Create your own Watson IoT Instance in your Bluemix account](#create-your-own-watson-iot-instance-in-your-bluemix-account)
1. [Create a sample Watson IoT NodeRED application](#create-a-sample-watson-iot-nodered-application)
1. [Bind the Watson NodeRED application to your Watson IoT instance](#bind-the-watson-nodered-application-to-your-watson-iot-instance)
1. [Configure the Watson IoT ARM mbed Cloud Connect Bridge](#configure-the-watson-iot-arm-mbed-cloud-connect-bridge)
1. [Import the NodeRED Flow example](#import-the-nodered-flow-example)
1. [Configure your new Watson IoT Application Node Flow](#configure-your-new-watson-iot-application-node-flow)
1. [Finally, check that it all works](#finally-check-that-it-all-works)

### What you need

#### Device

An mbed device. In this tutorial, we use the [K64F development board](https://developer.mbed.org/platforms/FRDM-K64F/).

#### Accounts

- Create an account with [mbed developer](https://developer.mbed.org).
- Using your mbed developer credentials, check that you can log into [mbed Cloud Connect](https://connector.mbed.com).
- Create and set up an IBM Bluemix account:
    - Go to the [Bluemix Dashboard](https://console.ng.bluemix.net/).
    - Click **Sign up**.
    - Complete the form and click **Create account**.
    - A confirmation email is sent to the email address you specified. It must be acknowledged.
    - Log into your Bluemix account and specify the default organization and space.

#### Tools

##### Windows

* If you are working on a Windows version earlier than Windows 10, please install the [mbed USB Serial driver](https://developer.mbed.org/media/downloads/drivers/mbedWinSerial_16466.exe):

	**Important:** Connect the mbed device over USB cable *before* running the Serial Driver installation; the installer must be able to see the device throughout the whole process.
* Install a serial terminal, such as [PuTTY](http://www.putty.org/).
* Optional: you can also install the terminal application [CoolTerm](http://freeware.the-meiers.org/CoolTerm_Win.zip).
* Install either the Chrome or Firefox browsers. Note that IE will not work; you must use either Chrome or Firefox.

##### Mac

* Install a serial terminal such as [CoolTerm](http://freeware.the-meiers.org/CoolTerm_Mac.zip).

	**Note:** You might need to authorize the CoolTerm application under **System Preferences** >> **Security & Privacy**; allow apps to run from **any developer**, then authorize the launch of CoolTerm.
* Install either the Chrome or Firefox browsers.

##### Linux

* Install a serial terminal such as [CoolTerm](http://freeware.the-meiers.org/CoolTerm_Linux.zip).
* Install either the Chrome or Firefox browsers.

### Process

#### Log into the online IDE and import the K64F project

This example is a clean and simple mbed endpoint which exposes two CoAP resources: accelerometer and LED.

1. Go to [the mbed Online Compiler (online IDE)](https://developer.mbed.org/compiler/).
1. Right-click **My Programs**, and select **Import Program** > **From URL…**
1. Enter the following URL: `https://github.com/ARMmbed/mbed-ethernet-sample-techcon2016/`
1. Leave the **Update all libraries to the latest revision** box unchecked. 
1. Click **Import**. 
1. When the import finishes, you can review the sample project's code.
1. The mbed Online Compiler needs to know which device it's compiling for; check that **FRDM-K64F** is selected:
    1. You can see your selected device in the right-hand corner of the compiler. 
    1. If you see a different device, click the device to open the **Select a Platform** window and select the FRDM-K64F. 
    1. If the FRDM-K64F is not available for selection, add it to your compiler by clicking the **Open mbed Compiler** button [on the K64F page](https://developer.mbed.org/platforms/FRDM-K64F/).

#### Create your mbed Cloud Connect Access Key

1. Open the mbed Cloud Connect Dashboard: [https://connector.mbed.com](https://connector.mbed.com)
1. Log in.
1. From the left-side menu, select **My applications** >> **Access Keys**.
1. Click **Create new access key**.
1. Give the key a name.
1. Save the key. You will need it to configure the Watson IoT ARM mbed Cloud Connect Bridge.

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/access_key.png)</span>


#### Set the developer credentials in your endpoint code

1. Open the mbed Cloud Connect dashboard: [https://connector.mbed.com](https://connector.mbed.com)
1. In the left sidebar, select **Security Credentials** >> **Get My Security Credentials**.
1. Copy all contents of `security.h`. 
 
	**Note:** Make note of the `MBED_ENDPOINT_NAME` and `MBED_DOMAIN` values - you will use them to configure the Watson IoT ARM mbed Cloud Connect Bridge.
1. Go back to your online IDE session.
1. In the sample project, replace `security.h` with the the new `security.h` that you copied from the Connect dashboard.
1. Save.
1. Select the project name and click **Compile**. The endpoint code compiles for the K64F.
1. The online IDE downloads a `bin` file into your Downloads directory. Drag and drop this file to your `MBED` flash drive. 
   
   **Tip:** It may also be called `DAPLINK`.
1. The green LED on the board flickers for a bit, then stops. You may see the board dismount and remount. 
1. The sample project is now running on your board.

#### Run your endpoint code

1. Open your serial terminal:
    * CoolTerm:  
        1. Select: **Options** >> **Re-scan serial ports**. 
        1. Select the mbed one.
    * PuTTY: You must determine which COM port your mbed device is connected to. Check the Windows Device Manager.
1. For the endpoint serial configuration, set the baud rate to `115200` baud, and keep the defaults for everything else: `8`, `N`, `one`.
    
1. PuTTY: Select the **Serial** radio button.
1. Connect your serial terminal to the K64F:
    * CoolTerm: Click **Connect** (at the top of the window).
    * PuTTY: Click **Open**. 
1. Send a `Break` command from the serial terminal (or press the `RESET` button on the K64F):
    * CoolTerm: **Connection** >> **Send Break**
    * PuTTY: Right-click on the top of the window and select **Special Command** >> **Break**.
1. Check the output – you should see `endpoint registered`.

    CoolTerm output:

    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/endpoint_registered_coolterm.png)</span>

    PuTTY output:

    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/endpoint_registered_putty.png)</span>

#### Create your own Watson IoT instance in your Bluemix account

1. Open the Bluemix dashboard: [https://console.ng.bluemix.net](https://console.ng.bluemix.net)
1. Select **Services** >> **Internet of Things**.
1. Click **Get Started**.
1. Configure your Watson IoT instance.
1. From the *Connect* to drop-down list, select **Leave unbound**.
1. Scroll down and select the **Free** plan option.
1. Click **Create**
1. Your Watson IoT instance is created, and you should see the Watson IoT dashboard.
1. Click **Connect your devices** to open the dashboard.
1. Click **Apps** >> **Generate API Key**.
1. Make a note of the **API Key** and the **Authentication Token**. You will need them when you configure your application node flow.
1. Add a comment and click **Generate**.

#### Create a sample Watson IoT NodeRED application

1. Open the Bluemix dashboard: [https://console.ng.bluemix.net](https://console.ng.bluemix.net)
1. Click **Create Application**.
1. Click **NodeRED Starter**.
1. Enter an **App Name** (must be unique).
1. Make a note of your application URL. It has the following format: `<app name>.mybluemix.net`. You will need it when you import the NodeRED Flow example.
1. Click **Create**.
1. The application will stage (this might take a few minutes). 
1. When staging is complete, the application reports `app is running`.

#### Bind the Watson NodeRED application to your Watson IoT instance

The previous stage took you to the Watson dashboard:

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/watson_dashboard.png)</span>

1. Click **Connections** >> **Connect Existing**.
1. Click on your Watson IoT Instance badge, then **Connect**.
1. Your NodeRED application needs to restage (this might take a long time).
1. Wait until the state changes to `Your app is running`.

<span class="notes">**Note:** Keep this window open!</span>

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/watson_dashboard1.png)</span>

#### Configure the Watson IoT ARM mbed Cloud Connect Bridge

1. Open the Watson IoT Dashboard: [https://console.ng.bluemix.net](https://console.ng.bluemix.net)
1. Click **Extensions** (bottom icon) >> **Add Extension**.
1. For the **ARM mbed Connector** option, click **Add** >> **Setting up**.
1. Paste your mbed Cloud Connect Access key and `MBED_DOMAIN` values into the corresponding fields.
1. Click **Check Connection**.
1. If the connection is okay, click **Done**.

<span class="notes">**Note:** Keep this window open.</span>

#### Import the NodeRED Flow example

1. Open your Online IDE workspace.
1. Open your `mbed-ethernet-sample-techcon2016` project.
1. Double-click `NODEFLOW.txt` to open.
1. Copy the entire contents of the file.

	**Tip:** If you have trouble copying the ``NODEFLOW.txt`` file contents, try copying directly from the [file in project's GitHub repository](https://github.com/ARMmbed/mbed-ethernet-sample-techcon2016/blob/master/NODEFLOW.txt).
 
1. Navigate to the URL that you recorded earlier for your Watson IoT NodeRED Application: `http://<app name>.mybluemix.net`
1. Select **go to your NodeRED flow editor**. 
1. Click the hamburger menu icon (far top-right) >> **Import** >> **Clipboard**.
1. Paste the JSON code from step 4 into the **Paste nodes here** window and click **OK**.

#### Configure your new Watson IoT Application Node Flow

In the diagram below, the input and output nodes are indicated in blue, and the command nodes are indicated in orange:

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/nodeflow_nodes.png)</span>

You need to link the blue and orange nodes to your Watson IoT instance. We will start with the blue nodes.

##### Prerequisites

You need the Watson API Key and Watson Auth Token that you got in the section [Create your own Watson IoT Instance in your Bluemix account](#create-your-own-watson-iot-instance-in-your-bluemix-account).

##### Procedure

1. Click the *accelerator observations* node.
1. Using the drop-down lists, set the following:
    * Authentication is your **API Key**.
    * API Key: 
		1. Click the edit (pencil) icon.
		1. Provide a name.
		1. Paste your Watson API Key and your Watson Auth Token into the relevant fields.
		1. Click **Update** to save.
    * Input Type is **Device Event**.
    * Device Type is **mbed-endpoint** or **all**.
    * Device ID can be found in `security.h`, under `MBED_ENDPOINT_NAME`.
    * Event is **notify** (for CoAP observations).
1. Click **Ok**.

    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/watson_configure.png)</span>

1. Click the *PUT* node.
1. Using the drop-down lists, set the following:
    * Authentication is **API Key**.
    * API Key is the same as above (for accelerator observation).
    * Input Type is **Device Command**.
    * Device Type is **mbed-endpoint** or **all**.
    * Device ID can be found in `security.h`, under `MBED_ENDPOINT_NAME`.
    * Event is **PUT** (all-caps).
1. Click **Ok**.

Lastly, we have to configure the remaining “Orange” command builder nodes. Note that each LED ON/OFF has a Base64 encoded value; no other encoding is supported. **1** is ON, **0** is OFF.

For both of the LED OFF/ON nodes:

1. Select the node and make a note the JSON payload created. 
1. Set the Device ID to be the same value as `MBED_ENDPOINT_NAME`.

	**Note:** The Device ID field is to the right of the JSON structure. Scroll or enlarge the window to see it.
1. Replace `MBED_ENDPOINT_NAME_GOES_HERE` with the actual `MBED_ENDPOINT_NAME` value.
1. Click **Done**. 
1. In the upper right of your NodeRED console, click the red **Deploy** button.

#### Final steps

##### Check the serial terminal

1. Press the **Reset** button on your mbed device (next to the USB port).
1. Open the **Debug** window in your NodeRED editor.
1. You should see output that looks something like this (make sure that you see a message like `endpoint registered`):
        
    CoolTerm output:

    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/endpoint_registered_coolterm.png)</span>

    PuTTY output:

    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/endpoint_registered_putty.png)</span>

##### NodeRED debug info

1. Open your Watson IoT application and NodeRED flow editor.
1. Open the debug window.
1. You should see output similar to that in the right column below.
  
You can toggle your LED on and off by clicking the **Turn LED OFF**/**ON** nodes. Look in the serial terminal for output from the device.

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/nodered_debug.png)</span>

##### Check Watson IoT devices

1. Open [your Watson dashboard](https://console.ng.bluemix.net).
1. Select **Devices**.
1. The bridge automatically creates Watson IoT devices for you.
1. Click on a device.
1. You can watch CoAP use your bridge to notify of events (observations):

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/watson_events.png)</span>
