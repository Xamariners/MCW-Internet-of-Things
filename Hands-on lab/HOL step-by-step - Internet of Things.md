![Microsoft Cloud Workshops](https://github.com/Microsoft/MCW-Template-Cloud-Workshop/raw/main/Media/ms-cloud-workshop.png 'Microsoft Cloud Workshops')

<div class="MCWHeader1">
Internet of Things
</div>

<div class="MCWHeader2">
Hands-on lab step-by-step
</div>

<div class="MCWHeader3">
June 2021
</div>

Information in this document, including URL and other Internet Web site references, is subject to change without notice. Unless otherwise noted, the example companies, organizations, products, domain names, e-mail addresses, logos, people, places, and events depicted herein are fictitious, and no association with any real company, organization, product, domain name, e-mail address, logo, person, place or event is intended or should be inferred. Complying with all applicable copyright laws is the responsibility of the user. Without limiting the rights under copyright, no part of this document may be reproduced, stored in or introduced into a retrieval system, or transmitted in any form or by any means (electronic, mechanical, photocopying, recording, or otherwise), or for any purpose, without the express written permission of Microsoft Corporation.

Microsoft may have patents, patent applications, trademarks, copyrights, or other intellectual property rights covering subject matter in this document. Except as expressly provided in any written license agreement from Microsoft, the furnishing of this document does not give you any license to these patents, trademarks, copyrights, or other intellectual property.

The names of manufacturers, products, or URLs are provided for informational purposes only and Microsoft makes no representations and warranties, either expressed, implied, or statutory, regarding these manufacturers or the use of the products with any Microsoft technologies. The inclusion of a manufacturer or product does not imply endorsement of Microsoft of the manufacturer or product. Links may be provided to third party sites. Such sites are not under the control of Microsoft and Microsoft is not responsible for the contents of any linked site or any link contained in a linked site, or any changes or updates to such sites. Microsoft is not responsible for webcasting or any other form of transmission received from any linked site. Microsoft is providing these links to you only as a convenience, and the inclusion of any link does not imply endorsement of Microsoft of the site or the products contained therein.

© 2021 Microsoft Corporation. All rights reserved.

Microsoft and the trademarks listed at <https://www.microsoft.com/en-us/legal/intellectualproperty/Trademarks/Usage/General.aspx> are trademarks of the Microsoft group of companies. All other trademarks are property of their respective owners.

**Contents**

- [Internet of Things hands-on lab step-by-step](#internet-of-things-hands-on-lab-step-by-step)
  - [Abstract and learning objectives](#abstract-and-learning-objectives)
  - [Overview](#overview)
  - [Solution architecture](#solution-architecture)
  - [Requirements](#requirements)
  - [Exercise 1: IoT Hub and Device Provisioning Service deployment](#exercise-1-iot-hub-and-device-provisioning-service-deployment)
    - [Task 1: Provision the IoT Hub](#task-1-provision-the-iot-hub)
    - [Task 2: Deploy the Device Provisioning Service](#task-2-deploy-the-device-provisioning-service)
    - [Task 3: Link the IoT Hub to the Device Provisioning Service](#task-3-link-the-iot-hub-to-the-device-provisioning-service)
    - [Task 4: Create an enrollment group](#task-4-create-an-enrollment-group)
  - [Exercise 2: Completing the Smart Meter Simulator](#exercise-2-completing-the-smart-meter-simulator)
    - [Task 1: Implement device management with the IoT Hub](#task-1-implement-device-management-with-the-iot-hub)
    - [Task 2: Configure the DPS Group Enrollment Key and ID Scope](#task-2-configure-the-dps-group-enrollment-key-and-id-scope)
    - [Task 3: Implement the communication of telemetry with IoT Hub](#task-3-implement-the-communication-of-telemetry-with-iot-hub)
    - [Task 4: Verify device registration and telemetry](#task-4-verify-device-registration-and-telemetry)
  - [Exercise 3: Sending commands to the IoT devices](#exercise-5-sending-commands-to-the-iot-devices)
    - [Task 1: Add your IoT Hub connection string to the CloudToDevice console app](#task-1-add-your-iot-hub-connection-string-to-the-cloudtodevice-console-app)
    - [Task 2: Send cloud-to-device messages](#task-2-send-cloud-to-device-messages)
  - [After the hands-on lab](#after-the-hands-on-lab)
    - [Task 1: Delete the resource group](#task-1-delete-the-resource-group)

# Internet of Things hands-on lab step-by-step

If you have not yet completed the steps to set up your environment in [Before the hands-on lab setup guide](./Before%20the%20HOL%20-%20Internet%20of%20Things.md), you will need to do that before proceeding.

## Abstract and learning objectives

In this hands-on lab, you will construct an end-to-end IoT solution simulating high velocity data emitted from smart meters and analyzed in Azure. You will design a lambda architecture, filtering a subset of the telemetry data for real-time visualization on the hot path, and storing all the data in long-term storage for the cold path.

At the end of this hands-on lab, you will be better able to build an IoT solution implementing device registration with the IoT Hub Device Provisioning Service and visualizing hot data with Power BI.

## Overview

Fabrikam provides services and smart meters for enterprise energy (electrical power) management. Their **You-Left-The-Light-On** service enables the enterprise to understand their energy consumption.

## Solution architecture

Below is a diagram of the solution architecture you will build in this lab. Please study this carefully, so you understand the whole of the solution as you are working on the various components.

![Diagram of the preferred solution described in the next paragraph.](./media/preferred-solution-architecture.png 'Preferred high-level architecture')

Smart Meters are installed in buildings. They will register with a Device Provisioning Service using an attestation method through an enrollment group. Once registered and connected, messages are ingested from the Smart Meters via the IoT Hub that the Device Provisioning Service assigned to the device. A Stream Analytics job pulls telemetry messages from IoT Hub and sends the messages to two different destinations. There are two Stream Analytics jobs, one that retrieves all messages and sends them to Blob Storage (the cold path), and another that selects out only the important events needed for reporting in real time (the hot path). Data entering the hot path will be reported on using Power BI visualizations and reports. For the cold path, Azure Databricks can be used to apply the batch computation needed for the reports at scale.

Other alternatives for processing of the ingested telemetry would be to use an HDInsight Storm cluster, a WebJob running the EventProcessorHost in place of Stream Analytics, or HDInsight running with Spark streaming. Depending on the type of message filtering being conducted for hot and cold stream separation, IoT Hub Message Routing might also be used, but this has the limitation that messages follow a single path, so with the current implementation, it would not be possible to send all messages to the cold path, while simultaneously sending some of the same messages into the hot path. An important limitation to keep in mind for Stream Analytics is that it is very restrictive on the format of the input data it can process: the payload must be UTF8 encoded JSON, UTF8 encoded CSV (fields delimited by commas, spaces, tabs, or vertical pipes), or AVRO, and it must be well-formed. If any devices transmitting telemetry cannot generate output in these formats (e.g., because they are legacy devices), or their output can be not well formed at times, then alternatives that can better deal with these situations should be investigated. Additionally, any custom code or logic cannot be embedded with Stream Analytics---if greater extensibility is required, the alternatives should be considered.

> **Note**: The preferred solution is only one of many possible, viable approaches.

## Requirements

- Microsoft Azure subscription must be pay-as-you-go or MSDN.
  - Trial subscriptions will not work.
- A virtual machine configured with:
  - Visual Studio Community 2019 or later
  - Azure SDK 2.9 or later (Included with Visual Studio)

## Exercise 1: IoT Hub and Device Provisioning Service deployment

Duration: 30 minutes

In your architecture design session with Fabrikam, it was agreed upon to use Azure Device Provisioning Service (DPS) to manage automatic device registration. The DPS would then assign an IoT Hub to the device that ingests telemetry from the Smart Meter Simulator. In this exercise, you will deploy an IoT Hub and DPS to enable device registration and connectivity.

### Task 1: Provision the IoT Hub

In these steps, you will provision an instance of IoT Hub.

1. In your browser, navigate to the [Azure portal](https://portal.azure.com), select **+Create a resource** in the navigation pane, enter `IoT Hub` into the **Search the Marketplace** box, and select **IoT Hub** from the results.

   !["IoT Hub" is entered into the Search the Marketplace box. IoT Hub is highlighted in the search results.](./media/create-resource-iot-hub.png 'Create an IoT Hub')

2. On the resource overview page, select **Create**.

3. On the **IoT Hub** screen, **Basics** tab, enter the following:

   - **Subscription**: Select the subscription you are using for this hands-on lab.

   - **Resource group**: Choose Use existing and select the **hands-on-lab-SUFFIX** resource group.

   - **Region**: Select the location you are using for this hands-on lab.

   - **IoT Hub Name**: Enter a unique name, such as `smartmeter-hub-SUFFIX`.

     ![The Basics tab for IoT Hub is displayed, with the values specified above entered into the appropriate fields.](./media/iot-hub-basics-blade.png 'Create IoT Hub Basics tab')

4. Select the **Management** tab. Accept the default Pricing and scale tier of **S1: Standard tier**, and select **Review + create**.

    ![The Management tab for IoT Hub is displayed with the Standard pricing tier selected.](media/iot-hub-management-tab.png 'Create IoT Hub Management tab')

5. Once validation has passed, select **Create**.

6. When the IoT Hub deployment is completed, you will receive a notification in the Azure portal. Select **Go to resource** in the notification.

   ![Screenshot of the Deployment succeeded message, with the Go to resource button highlighted.](./media/iot-hub-deployment-succeeded.png 'Deployment succeeded message')

7. From the **IoT Hub's Overview** blade, select **Shared access policies** under **Settings** on the left-hand menu.

   ![Screenshot of the Overview blade, settings section. Under Settings, Shared access policies is highlighted.](./media/iot-hub-shared-access-policies.png 'Overview blade, settings section')

8. Select **iothubowner** policy.

   ![The Azure portal is shown with the iothubowner selected.](./media/iot-hub-shared-access-policies-iothubowner.png 'IoT Hub Owner shared access policy')

9. In the **iothubowner** blade, select the **Copy** button to the right of the **Connection string - primary key** field. Record this value for a future task.

   ![Screenshot of the iothubowner blade. The connection string - primary key field is highlighted.](./media/iot-hub-shared-access-policies-iothubowner-blade.png 'iothubowner blade')

### Task 2: Deploy the Device Provisioning Service

In these steps, you will deploy an instance of the Device Provisioning Service (DPS).

1. In your browser, navigate to the [Azure portal](https://portal.azure.com), select **+Create a resource** in the navigation pane, enter `IOT Hub Device Provisioning Service` into the **Search the Marketplace** box, and select **IoT Hub** from the results.

    ![The Search the Marketplace textbox is shown with IoT Hub Device Provisioning Service entered as the search criteria and is selected from the search results.](media/dps_marketplace_search.png "Search the Marketplace")

2. On the resource overview page, select **Create**.

3. On the IoT Hub device provisioning service **Basics** tab, complete the form as follows:

   - **Subscription**: Select the subscription you are using for this hands-on lab.

   - **Resource group**: Choose Use existing and select the **hands-on-lab-SUFFIX** resource group.

   - **Name**: Enter a unique name, such as `smartmeter-dps-SUFFIX`.

   - **Region**: Select the location you are using for this hands-on lab.

    ![The DPS creation form Basics tab is shown populated with the above values.](media/dps_basics_form.png "DPS Basics Tab")

4. Select **Review + create**, then once validation has passed, select **Create** once more to deploy the service.

5. When the DPS deployment is completed, select **Go to resource** on the deployment screen.

6. On the Overview screen of the Device Provisioning Service, copy and record the value for **ID Scope** for a future task.

    ![The DPS Overview screen displays with the ID Scope field highlighted.](media/dps_overview_essentials.png "DPS Overview screen.")

### Task 3: Link the IoT Hub to the Device Provisioning Service

1. Remaining in the DPS resource, select **Linked IoT hubs** from the left menu, located beneath the **Settings** heading. Then select **+Add** from the toolbar.

    ![The DPS Linked IoT hubs screen is shown with Linked IoT hubs selected in the left menu and the +Add button highlighted in the toolbar.](media/dps_linkediothubs_add_menu.png "DPS Linked IoT hubs")

2. In the Add link to IoT hub blade, populate the form as follows, then select **Save**:

   - **Subscription**: Select the subscription you are using for this hands-on lab.
   - **IoT Hub**: Select the `smartmeter-hub-{SUFFIX}` IoT Hub.
   - **Access Policy**: Select `iothubowner`.

    ![The Add link to IoT hub blade is shown populated with the preceding values.](media/dps_addlinktoiothub_form.png "Add link to IoT hub blade")

### Task 4: Create an enrollment group

Creating an enrollment group enables Fabrikam to allow devices to self-register. This avoids the need to register each device manually. Group enrollments are made possible via secure Attestations, these could be via certificates or symmetric keys. In this example, we will use the symmetric key approach. Using symmetric keys should only be used in non-production scenarios, such as with this proof of concept.

1. Remaining in the DPS resource, select **Manage enrollments** from the left menu, then select **+Add enrollment group** from the toolbar menu.

    ![The DPS Manage enrollments screen displays with the Manage enrollments item selected from the left menu and the +Add enrollment group button highlighted on the toolbar.](media/dps_manageenrollments_menu.png "DPS Add enrollment group")

2. In the Add Enrollment Group form, populate it as follows, then select the **Save** button.

    - **Group name**: Enter `smartmeter-device-group`.
    - **Attestation Type**: Select `Symmetric Key`.
    - **Auto-generate keys**: Checked.
    - **IoT Edge device**: Select `False`.
    - **Select how you want to assign devices to hubs**: Select `Evenly weighted distribution`.
    - **Select the IoT hubs this group can be assigned to**: Select `smartmeter-hub-{SUFFIX}`.
    - **Select how you want the device data to be handled on re-provisioning**: Select `Re-provision and migrate data`.
    - **Initial Device Twin State**: Retain the default value.
    - **Enable entry**: Select `Enable`.
  
   ![The Add Enrollment Group form is shown populated with the preceding values.](media/dps_addenrollmentgroup_form.png "Add Enrollment Group form")

3. Select the newly created enrollment group from the **Enrollment Groups** list.

4. On the Enrollment Group Details screen, copy the **Primary Key** value and record it for a future task.

    ![The Enrollment Group Details screen is shown with the copy button highlighted next to the Primary Key field.](media/dps_enrollmentgroup_primarykey.png "Enrollment Group Details")

## Exercise 2: Completing the Smart Meter Simulator

Duration: 60 minutes

Fabrikam has left you a partially completed sample in the form of the Smart Meter Simulator solution. You will need to complete the missing lines of code that deal with device registration management and device telemetry transmission that communicate with your IoT Hub.

### Task 1: Implement device management with the IoT Hub

1. In **Visual Studio** on your **Lab VM**, use **Solution Explorer** to open the file `DeviceManager.cs`.

2. From the Visual Studio **View** menu, choose **Task List**.

   ![On the Visual Studio View menu, Task List is selected.](media/visual-studio-view-menu-task-list.png 'Visual Studio View menu')

3. In the **Task List**, you will see a list of **TODO** tasks, where each task represents one line of code that needs to be completed. Complete the line of code below each **TODO** using the code below as a reference. If your task list is blank, complete TODO steps 1-5 as indicated in the code in the next step.

4. The following code represents the completed tasks in **DeviceManager.cs**:

   ```csharp
    using System;
    using System.Security.Cryptography;
    using System.Text;
    using System.Threading.Tasks;
    using Microsoft.Azure.Devices.Provisioning.Client;
    using Microsoft.Azure.Devices.Provisioning.Client.Transport;
    using Microsoft.Azure.Devices.Shared;

    namespace SmartMeterSimulator
    {
        class DeviceManager
        {            
            /// <summary>
            /// Register a single device with the device provisioning service.
            /// </summary>
            /// <param name="enrollmentKey">Group Enrollment Key</param>
            /// <param name="idScope">DPS Service ID Scope</param>
            /// <param name="deviceId">Device Id of the device being registered</param>
            /// <returns></returns>
            public async static Task<SmartMeterDevice> RegisterDeviceAsync(string enrollmentKey, string idScope, string deviceId)
            {
                var globalEndpoint = "global.azure-devices-provisioning.net";
                SmartMeterDevice device = null;
                
                //TODO: 1. Derive a device key from a combination of the group enrollment key and the device id
                var primaryKey = ComputeDerivedSymmetricKey(enrollmentKey, deviceId);

                //TODO: 2. Create symmetric key with the generated primary key
                using (var security = new SecurityProviderSymmetricKey(deviceId, primaryKey, null))
                using (var transportHandler = new ProvisioningTransportHandlerMqtt())
                {
                    //TODO: 3. Create a Provisioning Device Client
                    var client = ProvisioningDeviceClient.Create(globalEndpoint, idScope, security, transportHandler);

                    //TODO: 4. Register the device using the symmetric key and MQTT
                    DeviceRegistrationResult result = await client.RegisterAsync();

                    //TODO: 5. Populate the device provisioning details
                    device = new SmartMeterDevice()
                    {
                        AuthenticationKey = primaryKey,
                        DeviceId = deviceId,
                        IoTHubHostName = result.AssignedHub
                    };
                }
                
                //return the device
                return device;
            }

            /// <summary>
            /// Compute a symmetric key for the provisioned device from the enrollment group symmetric key used in attestation.
            /// </summary>
            /// <param name="enrollmentKey">Enrollment group symmetric key.</param>
            /// <param name="deviceId">The device Id of the key to create.</param>
            /// <returns>The key for the specified device Id registration in the enrollment group.</returns>
            /// <seealso>
            /// https://docs.microsoft.com/en-us/azure/iot-edge/how-to-auto-provision-symmetric-keys?view=iotedge-2018-06#derive-a-device-key
            /// </seealso>
            private static string ComputeDerivedSymmetricKey(string enrollmentKey, string deviceId)
            {
                if (string.IsNullOrWhiteSpace(enrollmentKey))
                {
                    return enrollmentKey;
                }

                var key = "";
                using (var hmac = new HMACSHA256(Convert.FromBase64String(enrollmentKey)))
                {
                    key = Convert.ToBase64String(hmac.ComputeHash(Encoding.UTF8.GetBytes(deviceId)));
                }

                return key;
            }
        }

    }
   ```

   >**Note**:  Be sure you only replace code in the **DeviceManager** class and not any other code in the file.

5. Save **DeviceManager.cs**.

### Task 2: Configure the DPS Group Enrollment Key and ID Scope

You will want to avoid entering the DPS Group Enrollment Key and ID Scope every time the project is run. To do this, you can set this value as the default text for the `DPS Group Enrollment Primary Key` and `DPS ID Scope` text boxes in the application. Follow these steps to configure the connection string:

1. Return to the **SmartMeterSimulator** solution in **Visual Studio** on your **Lab VM**.

2. In the **Solution Explorer**, expand the **SmartMeterSimulator** project and double-click **MainForm.cs** to open it. (If the Solution Explorer is not in the upper-right corner of your Visual Studio instance, you can find it under the View menu in Visual Studio.)

   ![In the Visual Studio Solution Explorer window, SmartMeterSimulator project is expanded, and under it, MainForm.cs is highlighted.](media/visual-studio-solution-explorer-mainform-cs.png 'Visual Studio Solution Explorer')

    > **Note**: If the file does not open. One of the project files may be blocked.

   - Open **Windows Explorer** and navigate to the starter project folder: **C:\SmartMeter\Hands-on lab\lab-files\starter-project\SmartMeterSimulator**.
   - Right-click on the **MainForm.resx** file, then select **Properties**.
   - Check the **Unblock** checkbox on the bottom of the **General** tab, then select **Apply** then **OK**.

   ![Right-click MainForm.resx, go to Properties, then check the box next to Unblock](media/unblock-file.png 'Unblock file')

   - Close and reopen **Visual Studio**. Re-open the **MainForm.cs** file.

    >**Note**: If you are still unable to see the Windows Forms designer, close it, then right-click the project and select **Clean**. Then, right-click the project again and select **Build**. Now, you should be able to open the form without a problem.
    >
    >    ![Building and cleaning the solution to ensure that the Windows Forms editor shows up.](./media/build-and-clean-solution.png "Building and cleaning solution")
  
3. In the **Windows Forms designer surface**, select the **DPS Group Enrollment Primary Key** textbox.

   ![The Windows Form designer surface is opened to the MainForm.cs tab. The DPS Group Enrollment Primary Key textbox is highlighted, but is empty.](./media/smart-meter-simulator-iot-hub-connection-string.png 'Windows Form designer surface')

4. In the **Properties** panel, scroll until you see the **Text** property. Paste your **DPS Enrollment Group Primary Key** value copied from Exercise 1, Task 4, Step 4 into the value for the **Text** property. (If the properties window is not visible below the Solution Explorer, right-click the TextBox, and select **Properties**.)

   ![In the Properties panel, the Text property is highlighted, and is populated.](./media/smart-meter-simulator-iot-hub-connection-string-text-property.png 'Textbox properties')

5. Repeat steps 3 and 4 to populate the **DPS ID Scope** textbox with the value you recorded in Exercise 1, Task 2, Step 6.

6. Your DPS Group Enrollment Primary Key and ID Scope should now be present every time you run the **Smart Meter Simulator**.

   ![The Windows Form designer surface is opened to the MainForm.cs tab. The IoT Hub Connection String now displays.](./media/smart-meter-simulator-iot-hub-connection-string-populated.png 'IoT Hub Connection String dialog')

7. Save **MainForm.cs**.

### Task 3: Implement the communication of telemetry with IoT Hub

1. Open **Sensor.cs** from the **Solution Explorer**, and complete the **TODO** items 6 to 11 as indicated within the code that are responsible for transmitting telemetry data to the IoT Hub, as well as receiving data from IoT Hub.

2. The following code shows the completed result:

   ```csharp
    using System;
    using System.Text;
    using Microsoft.Azure.Devices.Client;
    using Newtonsoft.Json;


    namespace SmartMeterSimulator
    {
        /// <summary>
        /// A sensor represents a Smart Meter in the simulator.
        /// </summary>
        class Sensor
        {
            private DeviceClient DeviceClient;
            private string IotHubUri { get; set; }
            public string DeviceId { get; set; }
            public string DeviceKey { get; set; }
            public DeviceState State { get; set; }
            public string StatusWindow { get; set; }
            public string ReceivedMessage { get; set; }
            public double? ReceivedTemperatureSetting { get; set; }
            public double CurrentTemperature
            {
                get
                {
                    double avgTemperature = 70;
                    Random rand = new Random();
                    double currentTemperature = avgTemperature + rand.Next(-6, 6);

                    if (ReceivedTemperatureSetting.HasValue)
                    {
                        // If we received a cloud-to-device message that sets the temperature, override with the received value.
                        currentTemperature = ReceivedTemperatureSetting.Value;
                    }

                    if (currentTemperature <= 68)
                        TemperatureIndicator = SensorState.Cold;
                    else if (currentTemperature > 68 && currentTemperature < 72)
                        TemperatureIndicator = SensorState.Normal;
                    else if (currentTemperature >= 72)
                        TemperatureIndicator = SensorState.Hot;

                    return currentTemperature;
                }
            }
            public SensorState TemperatureIndicator { get; set; }

            public Sensor(string deviceId)
            {
                DeviceId = deviceId;

            }

            public void SetRegistrationInformation(string iotHubUri, string deviceKey)
            {
                IotHubUri = iotHubUri;
                DeviceKey = deviceKey;
                State = DeviceState.Registered;
            }
            public void InstallDevice(string statusWindow)
            {
                StatusWindow = statusWindow;
                State = DeviceState.Installed;
            }

            /// <summary>
            /// Connect a device to the IoT Hub by instantiating a DeviceClient for that Device by Id and Key.
            /// </summary>
            public void ConnectDevice()
            {
                //TODO: 6. Connect the Device to Iot Hub by creating an instance of DeviceClient
                DeviceClient = DeviceClient.Create(IotHubUri, new DeviceAuthenticationWithRegistrySymmetricKey(DeviceId, DeviceKey));

                //Set the Device State to Ready
                State = DeviceState.Connected;
            }

            public void DisconnectDevice()
            {
                //Delete the local device client            
                DeviceClient = null;

                //Set the Device State to Activate
                State = DeviceState.Registered;
            }

            /// <summary>
            /// Send a message to the IoT Hub from the Smart Meter device
            /// </summary>
            public async void SendMessageAsync()
            {
                var telemetryDataPoint = new
                {
                    id = DeviceId,
                    time = DateTime.UtcNow.ToString("o"),
                    temp = CurrentTemperature
                };

                //TODO: 7.Serialize the telemetryDataPoint to JSON
                var messageString = JsonConvert.SerializeObject(telemetryDataPoint);

                //TODO: 8.Encode the JSON string to ASCII as bytes and create new Message with the bytes
                var message = new Message(Encoding.ASCII.GetBytes(messageString));

                //TODO: 9.Send the message to the IoT Hub
                var sendEventAsync = DeviceClient?.SendEventAsync(message);
                if (sendEventAsync != null) await sendEventAsync;
            }

            /// <summary>
            /// Check for new messages sent to this device through IoT Hub.
            /// </summary>
            public async void ReceiveMessageAsync()
            {
                if (DeviceClient == null)
                    return;

                try
                {
                    Message receivedMessage = await DeviceClient?.ReceiveAsync();
                    if (receivedMessage == null)
                    {
                        ReceivedMessage = null;
                        return;
                    }

                    //TODO: 10.Set the received message for this sensor to the string value of the message byte array
                    ReceivedMessage = Encoding.ASCII.GetString(receivedMessage.GetBytes());
                    if (double.TryParse(ReceivedMessage, out var requestedTemperature))
                    {
                        ReceivedTemperatureSetting = requestedTemperature;
                    }
                    else
                    {
                        ReceivedTemperatureSetting = null;
                    }

                    // Send acknowledgement to IoT Hub that the has been successfully processed.
                    // The message can be safely removed from the device queue. If something happened
                    // that prevented the device app from completing the processing of the message,
                    // IoT Hub delivers it again.

                    //TODO: 11.Send acknowledgement to IoT hub that the message was processed
                    await DeviceClient?.CompleteAsync(receivedMessage);
                }
                catch (Exception)
                {
                    // The device client is null, likely due to it being disconnected since this method was called.
                    System.Diagnostics.Debug.WriteLine("The DeviceClient is null. This is likely due to it being disconnected since the ReceiveMessageAsync message was called.");
                }
            }
        }


        public enum DeviceState
        { 
            New,    
            Installed,
            Registered,         
            Connected,
            Transmit
        }
        public enum SensorState
        {
            Cold,
            Normal,
            Hot
        }
    }

   ```

    > **Note**:  Be sure you only replace the **Sensor** class and not any other code in the file.

3. Save **Sensor.cs**.

### Task 4: Verify device registration and telemetry

In this task, you will build and run the Smart Meter Simulator project.

1. In **Visual Studio** select **Build** from the Visual Studio menu, then select **Build Solution**.

2. Run the **Smart Meter Simulator**, by selecting the green **Start** button on the Visual Studio toolbar.

   ![The green Start button is highlighted on the Visual Studio toolbar.](media/visual-studio-toolbar-start.png 'Visual Studio toolbar')

3. Select one or more of the windows within the building to simulate the installation of a smart meter device. Once selected, the window will turn yellow.

    ![The smart meter simulator displays with three windows in yellow.](media/smart-meter-simulator-install-windows.png "The smart meter simulator")

4. Select **Register** on the **Smart Meter Simulator** window, this triggers the smart meter automatic registration through the enrollment group. It will take a few seconds for each of the yellow (installed) windows to turn to cyan, indicating the device is registered.

   ![The smart meter simulator window is shown with the previously yellow windows now displaying as cyan.](media/smart-meter-simulator-register.png 'Fabrikam Smart Meter Simulator')

5. At this point, you have installed and registered one or more devices (in cyan). To view this list of devices, you will switch over to the **Azure Portal**, and open the **IoT Hub** you provisioned.

6. From the **IoT Hub** blade, select **IoT Devices** under **Explorers** on the left-hand menu.

   ![On the IoT Hub blade, in the Explorers section, under Explorers, IoT Devices is highlighted.](media/iot-hub-explorers-iot-devices.png 'IoT Hub blade, Explorers section')

7. You should see all your registered devices listed having a status of **enabled**.

   ![Devices in the Device ID list have a status of either enabled or disabled.](media/iot-hub-iot-devices-list.png 'Device ID list')

8. In the **Azure Portal**, open the Device Provisioning Service resource, then select **Manage enrollments** from the left menu. Select the **smartmeter-device-group** enrollment group.

    ![The DPS Manage enrollments screen is shown with Manage enrollments selected in the left menu and the smartmeter-device-group highlighted in the enrollment groups listing.](media/dps_select_enrollmentgroup.png "Enrollment Groups Listing")

9. In the Enrollment Group Details screen, select the **Registration Records** tab and notice the devices selected for registration in the simulator application are listed.

    ![The DPS Enrollment Group Details screen displays with the Registration Records tab highlighted and a list of registered devices.](media/dps_enrollmentgroup_registrationrecords.png "Enrollment Group Registration Records")

10. Return to the **Smart Meter Simulator** window.

11. Select **Connect**. Within a few moments, you should begin to see activity as the windows change color, indicating the smart meters are transmitting telemetry. The grid on the left will list each telemetry message transmitted and the simulated temperature value.

    ![On the Smart Meter Simulator, the Connect button is highlighted, and one of the green windows has now turned to blue. The current windows count is now seven gray, two green, and one blue.](media/smart-meter-simulator-connect.png 'Fabrikam Smart Meter Simulator')

12. Allow the smart meter to continue to run for the duration of the lab.

## Exercise 5: Sending commands to the IoT devices

Duration: 20 minutes

Fabrikam would like to send commands to devices from the cloud in order to control their behavior. In this exercise, you will send commands that control the temperature settings of individual devices.

### Task 1: Add your IoT Hub connection string to the CloudToDevice console app

This console app is configured to connect to IoT Hub using the same connection string you use in the SmartMeterSimulator app. Messages are sent from the console app to IoT Hub, specifying a device by its ID, for example **Device1**. IoT Hub then transmits that message to the device when it is connected. This is called a **cloud-to-device message**. The console app is not directly connecting to the device and sending it the message. All messages flow through IoT Hub where the connections and device state are managed.

1. Return to the **SmartMeterSimulator** solution in **Visual Studio** on your **Lab VM**.

2. In the **Solution Explorer**, expand the **CloudToDevice** project and open **Program.cs**. (If the Solution Explorer is not in the upper-right corner of your Visual Studio instance, you can find it under the View menu in Visual Studio.)

    ![In the Visual Studio Solution Explorer window, CloudToDevice is expanded, and under it, Program.cs is highlighted.](media/visual-studio-solution-explorer-program-cs.png 'Visual Studio Solution Explorer')

3. Replace **YOUR-CONNECTION-STRING** on line 13 with your IoT Hub connection string. This is the same string you added to the Main form in the SmartMeterSimulator earlier. The line you need to update looks like this:

    ```csharp
    static string connectionString = "YOUR-CONNECTION-STRING";
    ```

    After updating, your **Program.cs** file should look similar to the following:

    ![The Program.cs file has been updated with the code change.](media/visual-studio-program-cs.png 'Program.cs')

4. Save the file.

### Task 2: Send cloud-to-device messages

In this task, you will leave the simulator running and separately launch the console app to start sending cloud-to-device messages.

1. If you hover over one of the windows, you will see a dialog display information about the associated device, including the Device ID (in this case, **Device0**), Device Key, Temperature, and Indicator. The legend on the bottom shows the indicator displayed for each temperature range. The Device ID is important when sending cloud-to-device messages, as this is how we will target a specific device when we remotely set the desired temperature. Keep the Device ID values in mind when sending the messages in the next task.

    ![A dialog containing device information is displayed after hovering over a window.](media/smart-meter-simulator-device-info.png 'Fabrikam Smart Meter Simulator')

2. Within the **SmartMeterSimulator** Visual Studio solution, right-click the **CloudToDevice** project, select **Debug**, then select **Start new instance** to run the console app.

3. In the **console window**, enter a **device number** when prompted. Accepted values are 0-9, since there are 10 devices whose IDs begin with 0. You can hover over the windows in the **Smart Meter Simulator** to view the Device IDs. When you enter a number, such as `0`, then a message will be sent to **Device0**. Be certain to select the device number of a registered window!

    ![The value of 0 is entered when prompted for the device number in the console window.](media/console-device-number.png 'Console App')

4. Now enter a **temperature value** between 65 and 85 degrees (F) when prompted. If you set a value above 72 degrees, the window will turn red. If the value is set between 68 and 72 degrees, it will turn green. Values below 68 degrees will turn the window blue. Once you set a value, the device will remain at that value until you set a new value, rather than randomly changing.

    ![A value of 70 has been entered for the temperature. A new log entry in the Smart Meter Simulator appears in yellow showing the message value of 70 sent to Device0.](media/console-temperature.png 'Console App and Smart Meter Simulator')

    If you run the **Smart Meter Simulator** side-by-side with the **console app**, you can observe the message logged by the Smart Meter Simulator within seconds. This message appears with a yellow background and displays the temperature request value sent to the device. In our case, we sent a request of 70 degrees to Device0. The console app indicates that it is sending the temperature request to the indicated device.

5. Hover over the device to which you sent the message. You will see that its temperature is set to the value you requested through the console app.

    ![Device0 is hovered over and the dialog appears showing the temperature set to the requested temperature.](media/smart-meter-simulator-set-temp.png 'Fabrikam Smart Meter Simulator')

6. In the console window, you can enter `Y` to send another message. Experiment with setting the temperature on other devices and observe the results.

## After the hands-on lab

Duration: 10 mins

In this exercise, you will delete any Azure resources that were created in support of the lab. You should follow all steps provided after attending the Hands-on lab to ensure your account does not continue to be charged for lab resources.

### Task 1: Delete the resource group

1. Using the [Azure portal](https://portal.azure.com), navigate to the Resource group you used throughout this hands-on lab by selecting Resource groups in the left menu.

2. Search for the name of your research group and select it from the list.

3. Select Delete in the command bar and confirm the deletion by re-typing the Resource group name, and selecting Delete.

You should follow all steps provided _after_ attending the Hands-on lab.
