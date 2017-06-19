### Connecting IoT devices to Microsoft Azure IoTHub with mbed Device Connector

#### Introduction

This tutorial explains how to connect your mbed device to Azure IoTHub, using mbed Device Connector and ARMs prototype IoTHub Connector Bridge.

Upon completion, you will be able to see and interact with your mbed devices from within the Azure IoTHub ecosystem.

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/azure_overview.png)</span>

#### What you need:

##### Devices
1. A Windows PC.
2. An mbed device. In this tutorial, we use the [K64F development board](https://developer.mbed.org/platforms/FRDM-K64F/).

##### Accounts

1. Create an account with [mbed developer](https://developer.mbed.org).
1. Using your mbed developer credentials, check that you can log in to [mbed Device Connector](https://connector.mbed.com).
1. Create and setup a Microsoft Azure account:
    - Go to the [Azure portal](https://www.azure.com).
    - Click **Free Account** > **Start for free**.
    - Complete the form and click **Sign up**.
        * A confirmation email is sent to the email address you specify. It must be acknowledged.
    - Log into your Azure account.

##### Tools

**IMPORTANT: You must use a Windows PC.**

1. Insert the USB cable and mbed device.
    * IMPORTANT: you must do this BEFORE running the Serial Driver installation.
1. Install the [mbed USB Serial driver](https://developer.mbed.org/media/downloads/drivers/mbedWinSerial_16466.exe).
    * You need administrator privilages to install this.
    * the installer MUST see the device first  
1. Install a serial Terminal, such as [PuTTY](http://www.putty.org/).
1. Install the [Docker Toolbox](https://github.com/docker/toolbox/releases/tag/v1.12.2
) on your Windows PC (v1.12.2 or later)
1. Install the latest version of [Microsoft Device Explorer](https://github.com/Azure/azure-iot-sdks/blob/master/tools/DeviceExplorer/doc/how_to_use_device_explorer.md). 
    * Note: Device Explorer may require additional .NET re-distributable downloads. The set-up installer will guide you. 
1. Optional: Install either the Chrome or Firefox browser. 


#### Process

1. First things first, connect a USB cable and Ethernet cable to your K64F board.

##### Log into the Online IDE and Import the K64F Project

1. Go to [https://developer.mbed.org](https://developer.mbed.org).
1. Click **Compiler** (top left).
1. Right-click **My Programs** >> **Import Program** >> **from URL…**.
    * NOTE: If this is your first time using the online compiler, a popup message prompts you to add a platform to your account. You need to do this.
        *  Click **Add platform**. 
        *  Select the **FRDM-K64F** board from the grid of options that appears. 
        *  Click **Add to my compiler**. 
        *  When it has been added, click **Open mbed compiler**. 
        *  In the popup that appears, leave the options as they are, and click **Open**.
1. In the **Source URL** field, enter: `https://github.com/ARMmbed/mbed-ethernet-sample-techcon2016/`
1. Keep the **Update all libraries to the latest revision** box unchecked.
1. Make sure **FRDM-K64F** is selected.
1. Click **Import**.   
 
<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/azure_import.png)</span>

##### Set the provisioning credentials in your endpoint code

1. Open the mbed Device Connector dashboard: [https://connector.mbed.com](https://connector.mbed.com)
1. In the left sidebar, click **Security Credentials** >> **Get My Security Credentials**
1. Copy all contents of `security.h`.
    * IMPORTANT: Make a note of the `MBED_ENDPOINT_NAME` and `MBED_DOMAIN` values.
1. Now, go back to the Compiler page of your online IDE
1. Replace `security.h` with the the content you copied from the Connect dashboard, and **Save**.
1. Open `main.cpp`, a clean and simple mbed endpoint example.
    * Two CoAP resources are exposed in this example; the accelerometer and LED.
1. Select the project name and click **Compile**.
1. The endpoint code should compile up successfully.
1. The online IDE will deposit a `bin` file into your downloads directory
1. Drag-n-Drop this bin file to your `MBED` flash drive 
    * NOTE: it may also be called `DAPLINK`.
1. The green LED on the K64F board will flicker and then stop, and then it will dismount and remount.

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/azure_provisioning.png)</span>

##### Run your endpoint code
1. Open your Serial Terminal.
    * PuTTY for Windows : Use Windows Device Manager to find the COM port of your mbed device.
1. For the endpoint serial configuration, set the baud rate to `115200` baud, defaults for everything else: `8`, `N`, `one`.
    * PuTTY for Windows: Make sure that the **Serial** radio button selected.
1. Connect your Serial Terminal to the K64F:
    * PuTTY for Windows: Click the **Open** button, and a terminal window will open.
1. Send a **Break** command from the serial terminal (or press the `RESET` button on the K64F)
    * PuTTY for Windows: Right-click on top of Window, and select **Special Command** >> **Break**
1. Look at the output – you need to confirm that you see `endpoint registered` in the output:


<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/azure_endpoint.png)</span>

##### Set-up the ARM IoTHub mbed Device Connector bridge

Before you begin, make sure you have the following tools installed and working:

* The latest Docker Toolbox runtime:
    * Ensure that **Git for Windows** is checked: [https://github.com/docker/toolbox/releases/tag/v1.12.2](https://github.com/docker/toolbox/releases/tag/v1.12.2)
* MS DeviceExplorer: [https://github.com/Azure/azure-iot-sdks/blob/master/tools/DeviceExplorer/doc/how_to_use_device_explorer.md](https://github.com/Azure/azure-iot-sdks/blob/master/tools/DeviceExplorer/doc/how_to_use_device_explorer.md)
* Windows mbed USB driver: 
    * `DAPLINK` flash drive present
    * `mbed Serial Port` seen in Windows Device Manager when K64F USB is connected
* PuTTY.

##### Create your mbed Device Connector access key

1. Open the mbed Cloud Connect dashboard and log in: [https://connector.mbed.com](https://connector.mbed.com)
1. In the left-hand menu, click **Access Keys**.
1. Click **Create New Access Key**, and give it a name
1. Save the key.

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/azure_api_token.png)</span>

##### Create your Azure IoTHub instance

1. Log into the Azure Portal: https://azure.microsoft.com
1. Search for "IoTHub".
1. Select the result and click **Create**.
1. Give your IoTHub a name.
1. Select the **Free** tier.
Azure will create the IoTHub instance.

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/azure_hub.png)</span>


##### Get your IoTHub Connection String

1. Open the IoTHub instance dashboard, and select your IoTHub resource.
1. Click **Settings**.
1. Select **Shared Access Policies** >> **iothubowner**.
1. Go to Connection String - Primary Key, and select Copy to Clipboard.

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/azure_string.png)</span>

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/azure_primary_string.png)</span>

##### Create your SAS Token with DeviceExplorer

1. Open DeviceExplorer.
1. Paste your connection string here.
1. Click **Update**.
1. In the **TTL (days)** field, enter `365`.
1. Click **Generate SAS**.
1. Copy ALL contents of your SAS Token, and save them.

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/azure_sas_token.png)</span>


##### Import the mbed Device Connector IoTHub bridge installer

1. Launch the Docker QuickStart Terminal
1. Enter the following `git` command:
    * `git clone https://github.com/ARMmbed/connector-bridge-container-installer`

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/azure_docker.png)</span>


##### Import the mbed Device Connector IoTHub bridge container

1. Go to the installer repo:
    * `cd connector-bridge-container-installer`
1. Run the installer script (look at options):
    * `./get_bridge.sh`
1. Run the installer script with options:
    * `% ./get_bridge.sh iothub use-long-polling`
The Bridge container will import from DockerHub.

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/azure_container.png)</span>

##### Configure your IoTHub bridge

Your container IP address is: `192.168.99.100`.

1. Open Firefox or Chrome, and go to: [https://192.168.99.100:8234](https://192.168.99.100:8234)
1. Accept the self signed certificate. The credentials are:
    * Username: `admin`
    * Password: `admin`
1. Enter the name of your IoTHub, and click **Save**.
1. Enter the Connector API Token, and click **Save**.
1. Enter your SAS Token, and click **Save**.
1. Click **Restart**.

Your IoTHub bridge is now configured.

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/azure_bridge.png)</span>

##### Check that it all works

1. Press the **reset** button on your mbed device (next to the USB port).
1. Look for `endpoint registered` in the serial terminal.
[image]
1. Open DeviceExplorer
1. Under the **Management** tab you should see your device listed
1. Under **Data** you should see output (if you do not see output, shake your device to generate some accelerometer readings).
1. Click **monitor** to start monitoring.

##### Interact with your device using DeviceExplorer

1. Open DeviceExplorer and click **Messages to Device**.
1. In the Message field, enter the following JSON, replacing `MBED_ENDPOINT_GOES_HERE` with your actual endpoint name:
`{"path":"/311/0/5850", "ep":"MBED_ENDPOINT_GOES_HERE", "new_value":"0","coap_verb":"put"}`
1. Click **Send**.

The blue LED should turn ON or OFF depending on `new_value` being `0` or `1`.

<span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/azure_interact.png)</span>

