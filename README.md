# Provisioning the ATSAME54-XPRO Development Kit for Microsoft Azure IoT Central

This user guide describes how to connect the [SAM E54 Xplained Pro Evaluation Kit](https://www.microchip.com/en-us/development-tool/atsame54-xpro) (Part No. [ATSAME54-XPRO](https://www.microchipdirect.com/dev-tools/ATSAME54-XPRO?allDevTools=true)) to [Azure IoT Central](https://docs.microsoft.com/en-us/azure/iot-central/core/overview-iot-central). The provided firmware project leverages Microsoft’s [Azure RTOS™](https://azure.microsoft.com/en-us/products/rtos/) to enable better experiences of embedded firmware development for Cloud applications. Download the [SAM E54 Xplained Pro User Guide](https://www.microchip.com/content/dam/mchp/documents/OTH/ProductDocuments/UserGuides/70005321A.pdf) and the [SAM E54 Xplained Pro Design Documentation](https://ww1.microchip.com/downloads/en/DeviceDoc/SAM-E54-Xplained-Pro-Design-Documentaion-rev9.zip) for more details including the schematics for the board.
 
 <img src=".//media/ATSAME54-XPRO.png"/>

## Table of Contents

- [Additional Hardware Requirements](#additional-hardware-requirements)
- [Background Knowledge](#background-knowledge)
  - [Microchip “Provisioning” vs. Microsoft “Provisioning”](#microchip-provisioning-vs-microsoft-provisioning)
  - [TLS Connection](#tls-connection)
  - [MQTT Connection](#mqtt-connection)
- [Create an Azure Account and Subscription](#create-an-azure-account-and-subscription)
- [Program the ATSAME54-XPRO Development Board](#program-the-atsame54-xpro-development-board)
  - [1. Install the Development Tools](#1-install-the-development-tools)
  - [2. Connecting to Azure IoT Central](#2-connecting-to-azure-iot-central)
  - [3. Reading the ATECC608B Certificate Information](#3-reading-the-atecc608b-certificate-information)
  - [4. Configuring the Embedded Application](#4-configuring-the-embedded-application)
  - [5. Create the IoT Central Application](#5-create-the-iot-central-application)
  - [6. Create a New Device in the IoT Central Application](#6-create-a-new-device-in-the-iot-central-application)
  - [7. Update the ID Scope in the Project](#7-update-the-id-scope-in-the-project)
  - [8. Enable the Weather Click for Temperature Measurements](#8-enable-the-weather-click-for-temperature-measurements)
  - [9. Reporting Additional Weather Telemetry](#9-reporting-additional-weather-telemetry)
- [References](#references)
- [Conclusion](#conclusion)

## Additional Hardware Requirements

 The ATSAME54-XPRO development board connects to the Internet using a standard RJ45 Ethernet cable, so a direct connection to an Ethernet router is required. In addition to the board's connection to a router, the following hardware components are required to successfully run the demonstration:

 - [ATECC608B TRUST](https://www.microchip.com/en-us/development-tool/DT100104)

    The ATECC608B TRUST is an add-on board for the CryptoAuth Trust Platform and other Microchip development platforms that contain a [mikroBUS™](https://www.mikroe.com/mikrobus) header. The board supports a [mikroElektronika](https://www.mikroe.com) header that connects to any board that has a mikroBUS™ connection. This board provides an alternative to the sample units that require a socket board for doing the initial development and testing.

    The ATECC608B TRUST contains the ATECC608B secure elements – [Trust&GO™](https://www.microchip.com/en-us/products/security/trust-platform/trust-and-go), [TrustFLEX™](https://www.microchip.com/en-us/products/security/trust-platform/trustflex) and [TrustCUSTOM™](https://www.microchip.com/en-us/products/security/trust-platform/trustcustom). This provides a user the ability to develop solutions with any of these secure elements based on the requirements of the application. The user guide provides a physical overview of the connections and switch settings implemented on the board. 

    <img src=".//media/ATECC608B_Trust_Board.jpeg" style="width:2.5in;height:1.in" alt="A screenshot of a cell phone Description automatically generated" />

 - [Weather Click](https://www.mikroe.com/weather-click)
 
    Weather click feature the BME280 integrated environmental unit from Bosch. It’s a sensor that detects humidity, pressure, and temperature, specifically designed for low current consumption and long-term stability. The click is designed to work on a 3.3V power supply. It communicates with the target microcontroller over SPI or I2C interface.

    <img src=".//media/Weather_Click.png" style="width:2.0in;height:2.0in" alt="A screenshot of a cell phone Description automatically generated" />

 - [mikroBUS™ Xplained Pro](https://www.microchip.com/en-us/development-tool/ATMBUSADAPTER-XPRO) (QTY = 2)

    The mikroBUS™ Xplained Pro (XPRO) is an extension board in the Xplained Pro evaluation platform. It is designed to demonstrate mikroBUS™ click boards with Xplained Pro MCU boards. Two of these extension boards are required to run the full demonstration (one each for the ATECC608B TRUST & Weather click boards).

    <img src=".//media/mikroBUS_XPRO.png" style="width:2.0in;height:1.0in" alt="A screenshot of a cell phone Description automatically generated" />

Using the two mikroBUS™ Xplained Pro extension boards, connect the `ATECC608B Trust` and `Weather click` boards to the ATSAME54-XPRO's `EXT1` & `EXT2` expansion headers, respectively.

Confirm that the dip switch and jumper settings are correct for the ATECC608B Trust and its associated XPRO extension board underneath it:

<img src=".//media/ATECC608B_Trust_Settings.png" />

Complete the hardware setup by connecting micro-USB & Ethernet cables to their respective connectors as shown. Connect the board to the PC using the USB connector (located in one corner of the board) labeled `USB DEBUG` (note there are 2 different-labeled USB connectors on the board). The Ethernet cable needs to be plugged directly into the RJ45 jack of any standard switch/hub/router with Internet connectivity.

<img src=".//media/Picture11.jpg" />

## Background Knowledge

### Microchip “Provisioning” vs. Microsoft “Provisioning”

The term “provisioning” is used throughout this document (e.g. provisioning key, provisioning device, Device Provisioning Service). On the Microchip side, the provisioning process is to securely inject certificates into the hardware. From the context of Microsoft, provisioning is defined as the relationship between the hardware and the Cloud (Azure). [Azure IoT Hub Device Provisioning Service (DPS)](https://docs.microsoft.com/azure/iot-dps/#:~:text=The%20IoT%20Hub%20Device%20Provisioning%20Service%20%28DPS%29%20is,of%20devices%20in%20a%20secure%20and%20scalable%20manner.)
allows the hardware to be provisioned securely to the right IoT Hub.

### High Level Architecture between the Client (Microchip ATSAME54-XPRO) and the Cloud (Microsoft Azure)

This high-level architecture description summarizes the interactions between the ATSAME54-XPRO Development Board and Azure. These are the major puzzle pieces that make up this enablement work of connecting ATSAME54-XPRO Development Board to Azure through DPS using the most secure authentication:

- [Trust&GO™ Platform](https://www.microchip.com/en-us/products/security/trust-platform/trust-and-go): Microchip-provided implementation for secure authentication.  Each Trust&GO™ secure element comes with a pre-established locked configuration for thumbprint authentication and keys, streamlining the process of enabling network authentication using the [ATECC608B](https://www.microchip.com/en-us/product/ATECC608B) secure element. 

- [Azure RTOS™](https://azure.microsoft.com/en-us/services/rtos/): Microsoft-provided API designed to allow small, low-cost embedded IoT devices to communicate with Azure services, serving as translation logic between the application code and transport client

- [Azure IoT Central](https://docs.microsoft.com/en-us/azure/iot-central/core/overview-iot-central): IoT Central is an IoT application platform that reduces the burden and cost of developing, managing, and maintaining enterprise-grade IoT solutions

- [Azure IoT Hub](https://docs.microsoft.com/en-us/azure/iot-hub/about-iot-hub): IoT Hub is a managed service, hosted in the Cloud, that acts as a central message hub for bi-directional communication between your IoT application and the devices it manages

- [Device Provisioning Service (DPS)](https://docs.microsoft.com/en-us/azure/iot-dps/): a helper service for IoT Hub that enables zero-touch, just-in-time provisioning to the right IoT Hub without requiring human intervention, allowing customers to automatically provision millions of devices in a secure and scalable manner

- [Public Key Cryptography Standards number 11 (PKCS#11)](https://www.microchip.com/en-us/about/media-center/blog/2022/atecc608-trustflex-secure-element): an interface to trigger cryptographic operations that will leverage secrets (the keys). In simple terms, it is a standard interface between an operating system and a Hardware Security Module (HSM), which in this case is the ATECC608B secure element and Azure RTOS. Microsoft Azure has conveniently integrated the PKCS#11 interface inside its Azure RTOS. Cryptographic commands will go through Azure RTOS to PKCS#11 but there is an intermediate library needed: Microchip [CryptoAuthLib](https://www.microchip.com/CryptoAuthlib). This library makes the secure element agnostic of the MCU or processor. CryptoAuthLib already supports calls from a PKCS#11 interface and translates them into low-level commands to the ATECC608B TrustFLEX or TA100 secure element as shown in the graphic below.

    <img src=".//media/Cryptoauthlib-Block-Diagram.png" />

On successful authentication, the ATSAME54-XPRO Development Board will be provisioned to the correct IoT Hub that is pre-linked to DPS during the setup process. We can then leverage Azure IoT Central's application platform services (easy-to-use, highly intuitive web-based graphical tools used for interacting with and testing your IoT devices at scale).

### TLS connection

The TLS connection performs both authentication and encryption.
Authentication consists of two parts:

- Authentication of the server (the device authenticates the server)
- Authentication of the client (the server authenticates the device)

Server authentication happens transparently to the user since this demo uses Microchip's Trust&GO secure element ([ATECC608B](https://www.microchip.com/en-us/product/ATECC608B)) which comes preloaded with the required CA certificate. During client authentication the client private key must be used, but since this is stored inside the secure element and cannot be extracted, all calculations must be done inside the secure element. The main application will in turn call the secure element's library API’s to perform the calculations. Before the TLS connection is complete, a shared secret key must be negotiated between the server and the client. This key is used to encrypt all future communications during the connection.

### MQTT Connection

After successfully connecting on the TLS level, the board starts establishing the MQTT connection. Since the TLS handles authentication and security, MQTT does not require a username nor password.

## Create an Azure Account and Subscription

Before connecting to Azure, you must first create a user account with a valid subscription. The Azure free account includes free access to popular Azure products for 12 months, $200 USD credit to spend for the first 30 days, and access to more than 25 products that are always free. This is an excellent way for new users to get started and explore.

To sign up, you need to have a phone number, a credit card, and a Microsoft or GitHub account. Credit card information is used for identity verification only. You won't be charged for any services unless you upgrade. Starting is free, plus you get $200 USD credit to spend during the first 30 days and free amounts of services. At the end of your first 30 days or after you spend your $200 USD credit (whichever comes first), you'll only pay for what you use beyond the free monthly amounts of services. To keep getting free services after 30 days, you can move to [pay-as-you-go](https://azure.microsoft.com/en-us/offers/ms-azr-0003p/) pricing. If you don't move to the **pay-as-you-go** plan, you can't purchase Azure services beyond your $200 USD credit — and eventually your account and services will be disabled. For additional details regarding the free account, check out the [Azure free account FAQs](https://azure.microsoft.com/en-us/free/free-account-faq/).

When you sign up, an Azure subscription is created by default. An Azure subscription is a logical container used to provision resources in Azure. It holds the details of all your resources like virtual machines (VMs), databases, and more. When you create an Azure resource like a VM, you identify the subscription it belongs to. As you use the VM, the usage of the VM is aggregated and billed monthly.  You can create multiple subscriptions for different purposes.

Sign up for a free Azure account for evaluation purposes by following the process outlined in the [Microsoft Azure online tutorial](https://docs.microsoft.com/en-us/learn/modules/create-an-azure-account/). It is highly recommended to go through the entire section of the tutorial so that you fully understand what billing and support plans are available and how they all work.

Should you encounter any issues with your account or subscription, [submit a technical support ticket](https://azure.microsoft.com/en-us/support/options/).

## Set Up the Out Of Box Demonstration

### 1. Install the Development Tools

Embedded software development tools from Microchip need to be pre-installed in order to properly program the ATSAME54-XPRO Development Board and provision it for use with Microsoft Azure IoT services.

- The Microchip `MPLAB X` tool chain for embedded code development on 32-bit architecture MCU/MPU platforms (made up of 3 major components which all need to be installed)

    (a) [MPLAB X IDE (minimum v6.05)](https://www.microchip.com/mplab/mplab-x-ide)

    (b) [MPLAB XC32 Compiler (minimum v4.20)](https://www.microchip.com/en-us/development-tools-tools-and-software/mplab-xc-compilers#tabs)

    NOTE: This demonstration project was tested successfully with XC32 v4.20, and in general should work with later versions of the compiler as they become available. If you encounter issues building the project, it is recommended to download XC32 v4.20 from the [MPLAB Development Ecosystem Downloads Archive](https://www.microchip.com/en-us/tools-resources/archives/mplab-ecosystem) (to fall back to the version Microchip successfully tested prior to release). 

    (c) [MPLAB Harmony Software Framework](https://microchipdeveloper.com/harmony3:mhc-overview)

- Any [Terminal Emulator](https://en.wikipedia.org/wiki/List_of_terminal_emulators) program of your choice

- Launch the terminal emulator program (e.g. PuTTY, TeraTerm, etc.) and connect to the COM port corresponding to your board at `115200 baud` (e.g. open PuTTY Configuration window &gt; choose `session` &gt; choose `Serial`&gt; enter/select the right COMx port). For example, you can find the right COM port number by opening your PC’s `Device Manager` &gt; expand `Ports(COM & LPT)` &gt; take note of `USB Serial Device (COMx)`

    <img src=".//media/image43.png"/>

### 2. Connecting to Azure IoT Central

Azure IoT technologies and services provide you with options to create a wide variety of IoT solutions that enable digital transformation for your organization. Use [Azure IoT Central](https://docs.microsoft.com/en-us/azure/iot-central/core/overview-iot-central), a managed IoT application platform, to build and deploy a secure, enterprise-grade IoT solution. IoT Central features a collection of industry-specific application templates, such as retail and healthcare, to accelerate your solution development processes.

[Azure IoT Central](https://docs.microsoft.com/en-us/azure/iot-central/core/overview-iot-central) is an IoT application platform that reduces the burden and cost of developing, managing, and maintaining enterprise-grade IoT solutions. Choosing to build with IoT Central gives you the opportunity to focus time, money, and energy on transforming your business with IoT data, rather than just maintaining and updating a complex and continually evolving IoT infrastructure.

The web UI lets you quickly connect devices, monitor device conditions, create rules, and manage millions of devices and their data throughout their life cycle. Furthermore, it enables you to act on device insights by extending IoT intelligence into line-of-business applications.

There are several ways to connect devices to Azure IoT. In this section, you learn how to connect a device by using Azure IoT Central. This user guide provide the steps to connect a single device to IoT Central using the individual enrollment method based on X.509 certification (with the ATECC608B’s device certificate).  This is the easiest way to develop a proof of concept that utilizes the full features of the ATECC608B secure element.

### 3. Reading the ATECC608B Certificate Information

To register a device on IoT Central using the X.509 individual enrollment method, we need two items that are contained within the ATECC608B.  We need the Common Name from the device certificate. This will be used as the device ID when we create a new device.  We also need the certificate saved as a *.PEM file, which is uploaded when we select an individual enrollment using X.509 certificates. The Azure RTOS demonstration application reads the certificate from the ATECC608B, and outputs this information on its debug serial port.

### 4. Configuring the Embedded Application using the MPLAB X IDE

- Download the ZIP file for this repository or clone it using GitHub tools (e.g. [Git Command Line](https://git-scm.com/book/en/v2/Getting-Started-The-Command-Line), [GitHub Desktop](https://desktop.github.com))

- Launch the MPLAB X IDE and open the provided example project `same54_iotc_demo.X` (located in the `firmware` sub-directory)

    <img src=".//media/Picture1a.png" width=300/>

    <img src=".//media/Picture1b.png" width=450/>

- Open the `sample_config.h` header file by double-clicking on the file in the `Projects` window

    <img src=".//media/Picture1c.png" width=250/>

- Search for the following definitions:

    ```bash
    #define ENABLE_ATECC608B 1
    #define USE_DEVICE_CERTIFICATE  1
    #define USE_THERMOSTAT_MODELID
    #define ENABLE_PRINT_CERTIFICATE 1
    #define ID_SCOPE
    //#define USE_WEATHER_CLICK
    ```

    The first time we run the application, the main considerations are the first four definitions above. These will configure the project to read the certificate and common name from the ATECC608B and print the information on the debug terminal emulator window.

    The project will have a pre-existing ID scope defined, but this value will need to be updated to the one used by your specific IoT Central application after it has been created.

    The `USE_WEATHER_CLICK` definition enables code that will communicate with a MikroBUS™ Weather click if it is installed on the `EXT2` expansion header.  This is disabled by default since many users will not have this board. It is recommended to leave this disabled at this point.  It can be enabled after the board is communicating with the IoT Central application.

- In the `Projects` view, configure the MPLAB X project by right-clicking on `same54_iotc_demo` and selecting `Properties` in the drop-down menu

    <img src=".//media/Picture1e.png" width=250/>

- Select `SAM E54 Xplained Pro` as the "Connected Hardware Tool" and click on the latest version of XC32 compiler that is currently installed. Click on `Apply` and then `OK`

    <img src=".//media/Picture1f.png" width=500/>

-  In the `Projects` window, right-click on the `same54_iotc_demo` project and select `Clean and Build` in the drop-down menu. The project should build without any error messages

    <img src=".//media/Picture1h.png" width=300/>

- Program the target board by clicking on the `Make and Program Device` icon in the MPLAB X main toolbar

    <img src=".//media/Picture1d.png" width=500/>

- Observe the messages in the MPLAB X `Output` window and confirm that the final message says `Programming complete`

    <img src=".//media/Picture1g.png" width=600/>

- Turn your focus to the terminal emulator window. You will see debug messages being generated as the demo program is going through its normal execution. Copy the device certificate information which is everything including the `-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----` lines. Using the text editor of your choice, paste the information into a newly-created file and save the file as `device_cert.pem`. The entire contents of the device certificate file should look something like the following:

    <img src=".//media/Picture2a.png" />

- Make a note of the `Common Name` that is output as one of the debug messages (typically this is based on a unique ID such as the device's serial number), which will also be used when we create/register the device in the IoT Central application. Once this information has been saved, you are ready to create your IoT Central Application!

    <img src=".//media/Picture2c.png" />

### 5. Create the IoT Central Application

- Sign into the Azure [IoT Central Portal](https://apps.azureiotcentral.com/home) and select `Build` on the side navigation menu

- Select `Create app` in the Custom app tile

    <img src=".//media/Picture4.png" />

- Add `Application Name` and a `URL` (suggest using the auto-generated defaults)

- Choose the pricing plan (recommend `Standard 2` as this allows the most messages and have 2 free devices connected for evaluation purposes)

    <img src=".//media/Picture5.png" />

- Select `Create`. After IoT Central provisions the application, it redirects you automatically to the new application which can be accessed in the future via the IoT Central portal or directly using the URL for the application

### 6. Create a New Device in the IoT Central Application

In this section, you use the IoT Central application to create a new device. You will use the connection information for the newly created device to securely connect your physical device in a later section. To create a device:

- From the application web page, select `Devices` on the side navigation menu.

- Select `Create a device` from the `All devices` pane to open the `Create a new device` window (if you're reusing an existing application that already has one or more devices, select `+ New` to open the window).

- Leave Device template as `Unassigned`. The device template will be assigned during the connection based on the the model ID used in the project.

- Set the `Device name` and `Device ID`.  The device name can be set freely, but the Device ID must be set to the common name of the X.509 certificate in the ATECC608B.

    <img src=".//media/Picture7.png" />

- Select the `Create` button. The newly created device will appear in the All devices list. 

- Select on the device name to show details.

- Select `Connect` on the top menu bar to configure the device connection parameters.

- On the connection screen, select `Individual enrollment` as the Authentication type and Certificates (X.509) as the Authentication method.

    <img src=".//media/Picture8.png" />

- For the primary certificate, click the folder icon and browse to the `device_cert.pem` file created earlier.  If the `Save` button is not enabled, upload the same `device_cert.pem` file for the secondary certificate.  The ATECC608B only has one certificate stored in it, but this workaround allows uploading the ATECC608B’s certificate to IoT Central. Note the ID scope for your application above (that will be used in the next step).

### 7. Update the ID Scope in the Project

- The final step is to update the `ID_SCOPE` definition in `sample_config.h` to match the ID scope from the connect screen above.  Rebuild the project and program the SAME54 Xplained Demo Board. After the board has been reset, you should see serial output showing a successful connection, followed by telemetry sending to IoT Central.

    <img src=".//media/Picture9a.png" />

- Go to the Device List in IoT Central and Select the device. You can view data being sent to the platform on the Raw Data Tab.  See example below.

    <img src=".//media/Picture10.png" />

### 8. Enable the Weather Click for Temperature Measurements

The base demonstration does not require a temperature click, but it will send relatively static temperature data to IoT Central.  If you have a weather click installed on `EXT2`, in the `sample_config.h` header file, uncomment the `USE_WEATHER_CLICK` macro, and rebuild.

<img src=".//media/Picture12.png" />

If the Weather Click is successfully communicating, you will see live ambient temperature readings reported to Azure.  If you apply heat to the click you can see the temperature rise.  It will report a new Max Temperature since last reboot any time the board sees a new high temperature.

<img src=".//media/Picture13.png" />

### 9. Reporting Additional Weather Telemetry

The Weather Click can also measure atmospheric pressure and relative humidity, but the standard thermostat device template we are using does not include those measurements in its device model.  Enable those measurements and sending them to the platform by commenting the `USE_THERMOSTAT_MODELID` definition.

  <img src=".//media/Picture14.png" />

When we disable the use of `USE_THERMOSTAT_MODELID`, the code has a spot for a new `MODEL_ID` to be inserted.  An example model ID is in the code, but this is not a publicly published model ID.

  <img src=".//media/Picture15.png" />

This was created using the IoT Central’s Device Template editor. To create your own Weather Click device template:

1.	Select Device templates from IoT Central’s left hand menu and click the “+ New” button.

2.	Select IoT Device, then hit the `Next: Customize` button at the bottom of the screen.

3.	Name your device template `MyWeatherClick`, and click “Next: Review”

4.	Select Custom Model. At this point there are a couple of ways you can proceed.  One way is to examine the Thermostat device template.  You can use the +Add capability button, and recreate the thermostat model using the graphical editor. This is a straightforward activity, albeit tedious. An easier way to enter the model is to leverage the DTDL model file that was created for the example, and edit the DTDL model in IoT Central's model editor.

5.	Click the `{} Edit DTL` button.  This will pull up a text editor with the default empty DTDL model. Your new device template, prior to adding in any capabilities will have a model that looks like this.  The model ID shown below in the `@id` field is unique to your application. This will eventually be put into the device project in `sample_config.h`.

    <img src=".//media/Picture16.png" />

6.	Replace everything below the top `@ID` line with elements that follow in the `WeatherClickModel.txt`.

7.	Update all the “@id” fields for each DTDL element, so they use your model ID.

    <img src=".//media/Picture18.png" />

8.	Once you have completed editing the DTDL file, save the changes. IoT Central will show you the graphical visualization of the new device template:

    <img src=".//media/Picture19.png" />

9.	Press the `Publish` button on the top menu. Once the model is published, your PnP application can work with the newly generated model.

10.	Update `sample_config.h` to point to the new Model ID.

    <img src=".//media/Picture20.png" />

11.	Rebuild the project and program the development board. When your project reconnects to IoT Central, it will declare it is using the MyWeatherClick device template by sending the new model ID when the connection is established. When you view the `All devices` list, you will see that your new device is using the MyWeatherClick device template. The project operates like the thermostat project, but it is now sending two additional telemetry elements (pressure and humidity). Select the device and view its `Raw Data`. The new telemetry will be continuously displayed in the list.

    <img src=".//media/Picture21.png" />

## References

Refer to the following links for additional information for IoT Explorer, IoT Hub, DPS, Plug and Play model, and IoT Central

•	[Manage cloud device messaging with Azure-IoT-Explorer](https://github.com/Azure/azure-iot-explorer/releases)

•	[Import the Plug and Play model](https://docs.microsoft.com/en-us/azure/iot-pnp/concepts-model-repository)

•	[Configure to connect to IoT Hub](https://docs.microsoft.com/en-us/azure/iot-pnp/quickstart-connect-device-c)

•	[How to use IoT Explorer to interact with the device](https://docs.microsoft.com/en-us/azure/iot-pnp/howto-use-iot-explorer#install-azure-iot-explorer)

•	[Azure IoT Central - All Documentation](https://docs.microsoft.com/en-us/azure/iot-central/)

•	[Create an Azure IoT Central application](https://docs.microsoft.com/en-us/azure/iot-central/core/quick-deploy-iot-central)

•	[Manage devices in your Azure IoT Central application](https://docs.microsoft.com/en-us/azure/iot-central/core/howto-manage-devices)

•	[How to connect devices with X.509 certificates for IoT Central](https://docs.microsoft.com/en-us/azure/iot-central/core/how-to-connect-devices-x509)

•	[Configure the IoT Central application dashboard](https://docs.microsoft.com/en-us/azure/iot-central/core/howto-add-tiles-to-your-dashboard)

•	[Customize the IoT Central UI](https://docs.microsoft.com/en-us/azure/iot-central/core/howto-customize-ui)

## Conclusion

You are now able to connect the ATSAME54-XPRO Development Board to Azure IoT Central and should have deeper knowledge of how all the pieces of the puzzle fit together between Microchip's hardware and Microsoft's Azure Cloud services. Let’s start thinking out of the box and see how you can apply this project to provision securely and quickly a massive number of Microchip devices to Azure and safely manage them through the entire product life cycle.