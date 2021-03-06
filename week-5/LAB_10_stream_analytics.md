# Lab 10 - Real-time stream processing with Stream Analytics

In this lab, you will learn how to process streaming data with Azure Stream Analytics. You will ingest vehicle telemetry data into Event Hubs, then process that data in real time, using various windowing functions in Azure Stream Analytics. They will output the data to Azure SQL Database.

After completing this lab, you will be able to:

- Use Stream Analytics to process real-time data from Event Hubs
- Use Stream Analytics windowing functions to build aggregates and output to SQL Database

## Technology overview

### Azure Stream Analytics

As more and more data is generated from a variety of connected devices and sensors, transforming this data into actionable insights and predictions in near real-time is now an operational necessity. [Azure Stream Analytics](https://docs.microsoft.com/azure/stream-analytics/stream-analytics-introduction) seamlessly integrates with your real-time application architecture to enable powerful, real-time analytics on your data no matter what the volume.

Azure Stream Analytics enables you to develop massively parallel Complex Event Processing (CEP) pipelines with simplicity. It allows you to author powerful, real-time analytics solutions using very simple, declarative [SQL like language](https://docs.microsoft.com/stream-analytics-query/stream-analytics-query-language-reference) with embedded support for temporal logic. Extensive array of [out-of-the-box connectors](https://docs.microsoft.com/azure/stream-analytics/stream-analytics-define-outputs), advanced debugging and job monitoring capabilities help keep costs down by significantly lowering the developer skills required. Additionally, Azure Stream Analytics is highly extensible through support for custom code with [JavaScript User Defined functions](https://docs.microsoft.com/azure/stream-analytics/stream-analytics-javascript-user-defined-functions) further extending the streaming logic written in SQL.

Getting started in seconds is easy with Azure Stream Analytics as there is no infrastructure to worry about, and no servers, virtual machines, or clusters to manage. You can instantly [scale-out the processing power](https://docs.microsoft.com/azure/stream-analytics/stream-analytics-streaming-unit-consumption) from one to hundreds of streaming units for any job. You only pay for the processing used per job.

[Guaranteed event delivery](https://docs.microsoft.com/stream-analytics-query/event-delivery-guarantees-azure-stream-analytics) and an enterprise grade SLA, provide the three 9's of availability, making sure that Azure Stream Analytics is suitable for mission critical workloads. Automated checkpoints enable fault tolerant operation with fast restarts with no data loss.

### Azure Event Hubs

[Azure Event Hubs](https://docs.microsoft.com/azure/event-hubs/event-hubs-about) is a big data pipeline that can ingest millions of events per second. It facilitates the capture, retention, and replay of telemetry and event stream data, using standard protocols such as HTTPS, AMQP, AMQP over websockets, and Kafka. The data can come from many concurrent sources and up to 20 consumer groups can allow applications to read entire event hub independently at their own pace.

## Scenario overview

Contoso Auto is collecting vehicle telemetry and wants to use Event Hubs to rapidly ingest and store the data in its raw form, then do some processing in near real-time. In the end, they want to create a dashboard that automatically updates with new data as it flows in after being processed. What they would like to see on the dashboard are various visualizations of detected anomalies, like engines overheating, abnormal oil pressure, and aggressive driving, using components such as a map to show anomalies related to cities, as well as various charts and graphs depicting this information in a clear way.

In this experience, you will use Azure Event Hubs to ingest streaming vehicle telemetry data as the entry point to a near real-time analytics pipeline built on Event Hubs, Azure Stream Analytics, and Azure SQL Database. Azure Stream Analytics extracts the vehicle sensor data from Event Hubs, performs aggregations over windows of time, then sends the aggregated data to Azure SQL Database for data analysis. A vehicle telemetry data generator will be used to send vehicle telemetry data to Event Hubs.

## Lab setup and pre-requisites

Before starting this lab, ensure you have the following services in your Azure environment:

- Databricks
- SQL Database (you can also work in Synapse - you can check out the instructions for Synapse [here](https://github.com/MicrosoftLearning/DP-203-Data-Engineer/blob/master/Instructions/Labs/LAB_10_stream_analytics.md))
  - In the instructions, we use a database called *telemetrydb*. If you have another name, that's fine. Just keep in mind to replace it when you encounter "our" database :-).
  - Basic edition (5 DTUs) will be enough for this lab
- Event Hubs (Standard pricing tier)
- Stream Analytics **job** (not *cluster*)
- Data Lake Storage Account

## Exercise 1 - Configure services

Azure Event Hubs is a Big Data streaming platform and event ingestion service, capable of receiving and processing millions of events per second. We are using it to temporarily store vehicle telemetry data that is processed and ready to be sent to the real-time dashboard. As data flows into Event Hubs, Azure Stream Analytics will query the data, applying aggregates and tagging anomalies, then send it to Azure SQL Database.

### Task 1: Configure Event Hubs

In this task, you will create and configure a new event hub within the provided Event Hubs namespace. This will be used to capture vehicle telemetry after it has been processed and enriched by the Azure function you will create later on.

1. Browse to the [Azure portal](https://portal.azure.com).

2. Select **Resource groups** where your Event Hubs is located.

3. Select the Event Hubs Namespace.

    ![The Event Hubs Namespace is selected in the resource group.](images/rg-event-hubs.png "resource group")

4. In the Event Hubs Namespace blade, select **Event Hubs** in the left-hand menu.

    ![The Event Hubs link is selected in the left-hand menu.](images/event-hubs-link.png 'Event Hubs link')

5. Create a new Event Hub called **telemetry**

6. Select the **telemetry** event hub.

    ![The newly created telemetry event hub is selected.](images/event-hubs-select.png 'Event hubs')

7. Select **Shared access policies** in the left-hand menu.

    ![The Shared access policies link is selected in the left-hand menu.](images/event-hubs-shared-access-policies-link.png 'Shared access policies link')

8. Select **+ Add** in the top toolbar to create a new shared access policy.

    ![The Add button is highlighted.](images/event-hubs-shared-access-policies-add-link.png 'Add')

9. In the **Add SAS Policy** blade, configure the following:

    - **Name:** `Read`
    - **Managed:** Unchecked
    - **Send:** Unchecked
    - **Listen:** Checked

        ![The Add SAS Policy form is filled out with the previously mentioned settings entered into the appropriate fields.](images/event-hubs-add-sas-policy-read.png 'Add SAS Policy')

        > It is a best practice to create separate policies for reading, writing, and managing events. This follows the principle of least privilege to prevent services and applications from performing unauthorized operations.

10. Select **Create** on the bottom of the form when you are finished entering the values.

11. Select **+ Add** in the top toolbar to create a second new shared access policy.

    ![The Add button is highlighted.](images/event-hubs-shared-access-policies-add-link.png 'Add')

12. In the **Add SAS Policy** blade, configure the following:

    - **Name:** `Write`
    - **Managed:** Unchecked
    - **Send:** Checked
    - **Listen:** Unchecked

        ![The Add SAS Policy form is filled out with the previously mentioned settings entered into the appropriate fields.](images/event-hubs-add-sas-policy-write.png 'Add SAS Policy')

12. Select **Create** on the bottom of the form when you are finished entering the values.

13. Select your **Write** policy from the list. Copy the **Connection string - primary key** value by selecting the Copy button to the right of the field. Save this value in Notepad or similar text editor for later.

    ![The Write policy is selected and its blade displayed. The Copy button next to the Connection string - primary key field is highlighted.](images/event-hubs-write-policy-key.png 'SAS Policy: Write')

### Task 2: Configure SQL Database

In this task, you will create a table in your SQL Database to store aggregate vehicle data provided by a Stream Analytics job that processes vehicle telemetry ingested by Event Hubs.

1. In the Azure Portal, open the database where the telemetry data has to land.

2. Connect with **Query editor**

3. Paste the following and select **Run** to create a new table:

    ```sql
    CREATE TABLE dbo.VehicleAverages
    ( 
        [AverageEngineTemperature] [float] NOT  NULL,
        [AverageSpeed] [float] NOT  NULL,
        [Snapshot] [datetime] NOT  NULL
    );
    ```

### Task 3: Configure Stream Analytics

Azure Stream Analytics is an event-processing engine that allows you to examine high volumes of data streaming from devices. Incoming data can be from devices, sensors, web sites, social media feeds, applications, and more. It also supports extracting information from data streams, identifying patterns, and relationships. You can then use these patterns to trigger other actions downstream, such as create alerts, feed information to a reporting tool, or store it for later use.

In this task, you will configure Stream Analytics to use the event hub you created as a source, query and analyze that data.

1. In the Azure portal, select the Stream Analytics job you configured for this lab.

    ![The Stream Analytics job is selected in the resource group.](images/rg-stream-analytics.png "resource group")

2. Within the Stream Analytics job, select **Storage account settings** in the left-hand menu, then select **Add storage account**.

    ![The storage account settings link and add storage account button are highlighted.](images/asa-storage-account.png "Storage account settings")

> This Storage Account is used to securely store all private "data assets" needed by this job. This includes job configuration, checkpoints and other necessary data required for the job to function correctly. So keep in mind: this is about configuration, not about the streaming or reference data.

3. In the **Storage account settings** form, configure the following:

   - **Select storage account from your subscriptions:** Selected.
   - **Subscription:** Make sure the subscription you are using for this lab is selected.
   - **Storage account:** Select the storage account you want to use in this lab
   - **Authentication mode:** Select "Connection string".

        ![The form is configured as described.](images/asa-storage-account-form.png "Storage account settings")

4. Select **Save**, then **Yes** when prompted to save the storage account settings.

5. Within the Stream Analytics job, select **Inputs** within the left-hand menu.

    ![The Inputs link is selected in the left-hand menu.](images/inputs-link.png 'Inputs link')

6. Select **+ Add stream input** in the top toolbar, then select **Event Hub** to create a new Event Hub input.

    ![The Add stream input button and Event Hub menu item are highlighted.](images/stream-analytics-add-input-link.png 'Add stream input - Event Hub')

7. In the **New Input** blade, configure the following:

    - **Name:** `eventhub`
    - **Select Event Hub from your subscriptions:** Selected
    - **Subscription:** Make sure the subscription you are using for this lab is selected.
    - **Event Hub namespace:** Select the Event Hub namespace you created in the preparation for this lab.
    - **Event Hub name:** Select **Use existing**, then select **telemetry**, which you created earlier.
    - **Event Hub consumer group:** Select **Use existing**, then select **$Default**.
    - **Authentication mode:** Select **Connection string**.
    - **Event Hub policy name:** Select **Use existing**, then select **Read**.
      - If the Event Hub policy "Read" is not yet visible, you might have to re-select the event hub "Telemetry" under the **Event Hub name** label
    - Leave all other values at their defaults.

        ![The New Input form is filled out with the previously mentioned settings entered into the appropriate fields.](images/stream-analytics-new-input.png 'New Input')

8. Select **Save** on the bottom of the form when you are finished entering the values.

9. Within the Stream Analytics job blade, select **Outputs** within the left-hand menu.

    ![The Outputs link is selected in the left-hand menu.](images/outputs-link.png 'Outputs link')

10. Select **+ Add** in the top toolbar, then select **SQL Database** to create a new SQL Database output.

11. In the **New Output** blade, configure the following:

    - **Output alias:** `sqldatabase`
    - **Select SQL Database from your subscriptions:** Selected.
    - **Subscription:** Select the subscription you are using for this lab.
    - **Database:** Select **telemetrydb**.
    - **Authentication mode:** Select **Connection string**.
    - **Username:**: Enter the SQL admin username of the database (select the database in the portal if you've forgotten the admin user). Default: `sqladmin`
    - **Password:** Enter the SQL admin password value you entered when deploying the SQL Database
    - **Server name** is already filled in (because you selected the correct database)
    - **Table:** `dbo.VehicleAverages`

12. Select **Save** on the bottom of the form when you are finished entering the values.

13. Within the Stream Analytics job blade, select **Query** within the left-hand menu.

    ![The Query link is selected in the left-hand menu.](images/query-link.png 'Query link')

14. Enter the following query:

    ```sql
    WITH
    VehicleAverages AS (
        select
            AVG(engineTemperature) averageEngineTemperature,
            AVG(speed) averageSpeed,
            System.TimeStamp() as snapshot
        FROM
            eventhub TIMESTAMP BY [timestamp]
        GROUP BY
            TumblingWindow(Duration(minute, 2))
    )
    -- INSERT INTO SQL DATABASE
    SELECT
        *
    INTO
        sqldatabase
    FROM
        VehicleAverages
    ```

    ![The query above has been inserted into the Query window.](images/stream-analytics-query.png 'Query window')

    The query averages the engine temperature and speed over a two second duration. The query aggregates the average engine temperature and speed of all vehicles over the past two minutes, using **TumblingWindow(Duration(minute, 2))**, and outputs these fields to the **sqldatabase** output.

15. Select **Save query** in the top toolbar when you are finished updating the query.

16. Within the Stream Analytics job blade, select **Overview** within the left-hand menu. On top of the Overview blade, select **Start**.

    ![The Start button is highlighted on top of the Overview blade.](images/stream-analytics-overview-start-button.png 'Overview')

19. In the Start job blade that appears, select **Now** for the job output start time, then select **Start**. This will start the Stream Analytics job so it will be ready to start processing and sending your events to Azure SQL Database.

    ![The Now and Start buttons are highlighted within the Start job blade.](images/stream-analytics-start-job.png 'Start job')

## Exercise 2 - Generate and aggregate data

### Task 1: Run data generator

The data generator console application creates and sends simulated vehicle sensor telemetry for an array of vehicles (denoted by VIN (vehicle identification number)) directly to Event Hubs. For this to happen, you first need to configure it with the Event Hub connection string.

In this task, you will configure and run the data generator. The data generator saves simulated vehicle telemetry data to Event Hubs, prompting your Stream Analytics job to aggregate and analyze the enriched data and send it to SQL Database.

1. Download the file [TransactionGenerator.zip](https://github.com/vstrien/DP-203-Data-Engineer/raw/master/Allfiles/TransactionGenerator.zip)
2. Extract the **TransactionGenerator.zip** archive to a subfolder named **TransactionGenerator**.
3. In the extracted **TransactionGenerator** folder, open the **appsettings.json** file. Paste your **telemetry** Event Hub connection string value next to **EVENT_HUB_CONNECTION_STRING**. Make sure you have quotes ("") around the value, as shown. **Save** the file.

    ![The Event Hub connection string is highlighted within the appsettings.json file.](images/appsettings.png "appsettings.json")

    > **Note:** Make sure that the connection string ends with *EntityPath=telemetry* (eg. *Endpoint=sb://YOUR_EVENTHUB_NAMESPACE.servicebus.windows.net/;SharedAccessKeyName=Write;SharedAccessKey=REDACTED/S/U=;EntityPath=telemetry*). If not, then you did not copy the connection string from the **Write** policy of your event hub.

    SECONDS_TO_LEAD is the amount of time to wait before sending vehicle telemetry data. Default value is 0.

    SECONDS_TO_RUN is the maximum amount of time to allow the generator to run before stopping transmission of data. The default value is 1800. Data will also stop transmitting when you enter Ctrl+C while the generator is running, or if you close the window.

4. In the extracted **TransactionGenerator** folder, run **TransactionGenerator.exe**.

5. If a **Windows protected your PC** dialog is displayed, select **More info**, then **Run anyway**.

    ![More info is highlighted.](images/microsoft-defender-moreinfo.png "Windows protected your PC")

    ![The Run anyway button is highlighted.](images/microsoft-defender-runanyway.png "Run anyway")

6.  A new console window will open, and you should see it start to send data after a few seconds. Once you see that it is sending data to Event Hubs, _minimize_ the window and keep it running in the background. Allow this to run for a mimimum of three minutes before moving onto the next step.

    ![Screenshot of the console window.](images/transaction-generator.png 'Console window')

    After every 500 records are requested to be sent, you will see output statistics.

### Task 2: View aggregate data in SQL Database

As you recall, when you created the query in Stream Analytics, you aggregated the engine temperature and vehicle speed data over two-minute intervals and saved the data to SQL Database. This capability demonstrates the Stream Analytics query's ability to write data to multiple outputs at varying intervals. Writing to a SQL Database enables us to retain the historic and current aggregate data as part of the data warehouse without requiring an ETL/ELT process.

In this task, you will view the anomaly data within SQL Database.


1. In the Azure Portal, open the database where the telemetry data has to land.

2. Connect with **Query editor**

3. Expand **Tables**, then right-click on the **dbo.VehicleAverages** table (f you do not see the table listed, refresh the tables list). Select **Select TOP 1000 rows**.

4. Run the query and view the results. Observe the aggregate data stored in **AverageEngineTemperature** and **AverageSpeed**. The **Snapshot** value changes in two-minute intervals between these records.

## Important: Cleanup

Complete these steps to stop the data generator and free up resources you no longer need.

### Task 1: Stop the data generator

1. Go back to the console/terminal window in which your data generator is running. Close the window to stop the generator.

### Task 2: Stop the Stream Analytics job

1. Navigate to the Stream Analytics job in the Azure portal.

2. In the Overview pane, select **Stop**, then select **Yes** when prompted.

    ![The stop button is highlighted.](images/asa-stop.png "Stop")
