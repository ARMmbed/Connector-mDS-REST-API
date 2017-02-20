## Connecting devices with AWS IoT and mbed Device Connector 

<span class="warnings">**Warning:** This tutorial uses prototypes and is not suitable for production.</span>

We're going to use mbed Connector and ARM's prototype AWS IoT Connector Bridge to connect an mbed device to AWS IoT. We'll:

1. Create a simple device and connect it to mbed Device Connector.
1. Create an instance of ARM's prototype AWS IoT Connector Bridge and bind it to your Amazon and Connector accounts.
1. Explore the device's data telemetry with Amazon's web-based MQTT client.

### What you need

__Accounts:__

* An [Amazon account](https://aws.amazon.com/console/). You can select the basic support option.
* An [mbed Developer account](https://developer.mbed.org/account/signup/?next=%2F).
* An [mbed Connector account](https://connector.mbed.com).

__Tools and drivers:__

* For Windows users: please install the [USB serial driver](https://developer.mbed.org/media/downloads/drivers/mbedWinSerial_16466.exe). To verify the driver was installed correctly, connect your device over USB and check that it mounts in the file system as either `MBED` or `DAPLINK`, and that Windows Device Manager shows `mbed Serial Port`.
* Serial terminal:
  * For Window: [PuTTy](http://www.putty.org/).
  * For Mac OS X: [CoolTerm](http://freeware.the-meiers.org/CoolTerm_Mac.zip).
    **Note:** You may have to authorize the application under **System preferences** > **Security and privacy**.
  * For Linux: [CoolTerm](http://freeware.the-meiers.org/CoolTerm_Linux.zip).
* Docker:
  * For [Windows](https://github.com/docker/toolbox/releases/tag/v1.12.2). When installing, please select **Custom installation** and verify that **Git for Windows** is selected:
  
    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/docker_with_git.png)</span>
    
  * For [Mac OS X](https://docs.docker.com/engine/installation/mac/). Install the native engine, not the toolbox version.
  * For [Linux](https://docs.docker.com/engine/installation/linux/).
* [Git](https://git-scm.com/).
* Chrome or Firefox.

__Device:__ 

NXP-FRDM-K64F. Please connect a USB and Ethernet cable to the device.

### Getting the sample application on your device

#### Adding the sample application to your mbed Online Compiler

To import the mbed sample project to the mbed Online Compiler:
   
1. Log into the [compiler](https://developer.mbed.org/compiler/).
1. In the **Program Workspace** navigation pane, right click on **My Programs**.
	
    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/import_program.png)</span>
	
1. Select **Import program** > **From URL**. The Import pop-up opens.
1. Enter ``https://github.com/ARMmbed/mbed-ethernet-sample-techcon2016``.
1. Leave the **Update all libraries** box unchecked.
	
    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/ethernet_sample_url.png)</span>
	
1. Click **Import**.
1. Verify that the selected device is FRDM-K64F
	
    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/selecting_platform.png)</span>
	
    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/frdm_k64f.png)</span>

#### Getting credentials for your sample application

1. Go to your [mbed Connector dashboard](https://connector.mbed.com).
1. In the left hand navigation pane, click **My devices** > **Security credentials**. The Security Credentials window opens.

    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/security_credentials_menu.png)</span>

1. Click **Get my device security credentials**. Content for a credential file is generated. 

    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/security_content.png)</span>

1. Copy the content.
1. Return to the mbed Online Compiler and your sample application.
1. The sample application has a file called `security.h`. Replace the content of this file with the content you copied from the mbed Connector dashboard.

    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/security_header.png)</span>

#### Running the application on your device

__Installing the application__

1. In the mbed Online Compiler, compile your application. The compiled `bin` file is downloaded to your `Downloads` folder.
1. Connect the device to your computer over USB. It will appear as either `MBED` or `DAPLINK`.
1. Copy the application onto your mbed device. The green LED flashes while the board is programmed. When the LED stops flashing, the board is ready.

__Running the applicaton__

1. Open your terminal application and find the board:
    * In CoolTerm, select **Options** > **Rescan serial ports**. Select the mbed USB option:
		
    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/serial_port.png)</span>

    * For PuTTy, check your Window Device Manager to determine your COM port and manually enter it in PuTTy's Configuration window:
		
    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/putty_serial_com.png)</span>
		
1. Set the baud rate to 115200. Leave all other options at default (8, N, one). 
1. If you're working with PuTTy on Windows, ensure you select the **Serial** radio-button pictured above.
1. Connect to your device:
    * In CoolTerm, click **Connect** from the top menu.
    * In PuTTy, click **Open** in the Configuration window.
1. Send the "break" command to reset the device:
    * In CoolTerm, select **Connection** > **Send Break**.
    * In PuTTy, right click on the window's top border and select **Special command** > **Break**. 
1. Monitor the device's output on your terminal (PSTOOI). You must be able to see ``endpoint registered``:
		
    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/endpoint_registered.png)</span>

### Setting up the Bridge

#### Creating an mbed Connector API token

1. Go to your [mbed Connector dashboard](https://connector.mbed.com).
1. In the left hand navigation pane, click **My applications** > **Access keys**. The Access Keys window opens.
		
    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/access_key_menu.png)</span>

1. Click **Create new access key**. 
		
    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/access_key_create.png)</span>

1. Enter a name for your access key.
		
    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/access_key_name.png)</span>

1. Click **Add**. An API token is created.
		
    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/access_key_token.png)</span>

1. Save the token.

#### Creating an AWS IoT instance

1. Log into the [AWS Management Console](https://aws.amazon.com/console/).
1. Click **AWS IoT**.

  ![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/aws_iot_in_menu.png)

1. If you do not have one already, create an Instance:

  ![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/aws_iot_get_started.png)

1. Open the Dashboard and click **Test**.

![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/aws_iot_dashboard.png)

#### Determining your AWS region

1. Determine your AWS Region from the Test Console URL:
    * Your URL should look something like this: 
`https://console.aws.amazon.com/iotv2/home?region=us-east-1#/test` 
    * Make a note of your "region" parameter value
        * In the above example, the AWS Region is `us-east-1`. 

#### Create your AWS Access Key and ID (New IAM User)

1. Using Chrome, go to: [http://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-setting-up.html](http://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-setting-up.html)
    * Complete the steps in Amazon's guide to create the Access Key and ID

1. If you do not have an Identity and Management (IAM) User, you need to create one:
    * Go to the AWS dashboard [link]
    * From the left hand menu, click **Users**.
    * Click **Add user**.
    * Enter a username.
    * Set Access type to **Programmatic Access**.
    * Click **Next Permissions** 

  ![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/aws_iot_user.png)

1. When the new IAM User is created, you will be shown your keys.
    * Click **Show** to see your Secret Access Key.
1. Save both keys.

  ![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/aws_iot_keys.png)

1. Click **Close**.


#### Create your AWS Access Key and ID (Existing IAM User)

If you have an existing IAM User, then you can simply create a new AccessKey for that user:

1. Go to the AWS dashboard: [link].

1. From the left hand menu, click **Users**.

1. Select your IAM User.

1. Select **Security Credentials**.

1. Click **Create Access Key**.

  ![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/aws_iot_existing_user.png)

1. Save both keys.    
   * Click **Show** to see your Secret Access Key.

  ![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/aws_iot_keys.png)

1. Click **Close**.


#### Verifying administrator permissions on IAM

Amazon's Identity and Access Management service (IAM) users should have administrator permissions to create *thing shadows*, which you will need later. 

1. In your AAWS Management Console, click **Users** 
1. Select your user.
1. Click **permissions**.
1. Check the **AdministratorAccess** policy box.
1. Click **Attach Policy**.

    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/aws_attach_policy.png)</span>

#### Importing the Connector AWS IoT bridge installer

1. Accessing a terminal:
    * On Windows: launch the Docker Quickstart Terminal. 

        **Note:** Please allow VirtualBox to make changes to your network interface.

    * On Mac OS X or Linux, open a terminal window.

1. Run the following git command: 

    ```
    $  git clone https://github.com/ARMmbed/connector-bridge-container-installer
    ```
	
1. `cd` into the installer repository:

    ```
    % cd connector-bridge-container-installer
    ```

1. Run the installer script:

    ```
    % ./get_bridge.sh
    ```
	
	You can run it with additional options, such as:
	
    ```
    % ./get_bridge.sh aws use-long-polling
    ```

1. Docker downloads the bridge container from Docker Hub. 

    <span class="notes">**Note:** When the script finishes running, it shows an IP address. If you're working on Mac OS X or Linux, please make a note of the IP address.</note>

#### Configuring the bridge 

1. Open Chrome or Firefox:
    * On Windows: go to `https://192.168.99.100:8234`.
    * On Mac OS X or Linux: please use the IP you got at the end of the Docker import script. Go to `https://<your container IP address>:8234`.
1. Accept the self signed certificate. 

    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/privacy_warning.png)</span>

1. Log in. Username: `admin`. Password: `admin`.
1. Enter the mbed Connector API token. Click **Save**.
1. Enter your AWS region. Click **Save**.
1. Enter your AWS access key. Click **Save**.
1. Enter your AWS access key ID. Click **Save**.
1. Click **Restart** at the top.

    <span class="images">![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/bridge_configuration.png)</span>

Your AWS bridge is now configured! You can restart your device and see messages coming into the AWS IoT via Amazon's MQTT Client.

### Putting it all together

#### Connecting to the device

1. Go to the AWS dashboard: [link]
1. In the left-hand menu, click **Registry** > **Things**.
1. You should see your device listed as a Thing.

  ![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/aws_iot_things.png)

1. In the left-hand menu, click **Test**.
    * Result: This opens the MQTT dashboard:

![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/aws_iot_test_dashboard.png)

##### Subscribe to the observation topic
1. In the subscription topic field, enter: `mbed/notify/#`.
1. Click **Subscribe to this topic**.

##### Subscribe to the command-response topci
1. In the subscription topic field, enter: `mbed/cmd-response/#`.
1. Click **Subscribe to this topic**.

  Result: 

  ![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/aws_iot_subscriptions.png)

1. Toggle between the messages by clicking on the topic names.

#### Interacting with the device

The application we installed on the device can receive a toggle that turns the LED on (`1`) and off (`0`). To trigger it, we will send a CoAP ``put`` command to the device.

![](https://s3-us-west-2.amazonaws.com/cloud-docs-images/aws_iot_coap.png)

1. In the field highlighted above, enter the following: `mbed/put/<endpoint_type>/<endpoint_name>/311/0/5850`

    <span class="tips">**Tip:** `<endpoint_type>` and `<endpoint_name>` are listed in the observations.</span>
    
2. In the terminal area below, enter the following: `{"path":"/311/0/5850", "ep":"ENDPOINT_NAME_GOES_HERE", "new_value":"0","coap_verb":"put"}`
    
    <span class="notes">**Note:** Make `ENDPOINT_NAME_GOES_HERE` your actual endpoint name and `"new_value":"0"` to turn the blue LED off.
  
1. Click **Publish to topic**.
1. Your blue LED should turn off (or on if you used `1`).
1. You will get a notification response in the `cmd-response` window that indicates that the CoAP `put` completed successfully!


