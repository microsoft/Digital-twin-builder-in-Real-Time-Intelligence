# Digital twin builder (preview) in Real-Time Intelligence lab part 2: Get streaming data

In this part of the lab, you set up another type of sample data: a real-time data stream of sample bus data that includes time series information about bus locations. You stream the sample data into an eventhouse, perform some transformations on the data, then create a shortcut to get the eventhouse data into the sample data lakehouse that you created in the previous section. Digital twin builder requires data to be in a lakehouse.

## Create an Eventhouse

[!INCLUDE [Real-Time Intelligence create-eventhouse](../lab/includes/create-eventhouse.md)]

## Create an Eventstream

In this section, you create an eventstream to send sample bus streaming data to the eventhouse.

### Add source

Follow these steps to create the eventstream and add the *Buses* sample data as the source.

1. From the **KQL databases** pane in the eventhouse, select the new **Tutorial** database.

2. From the menu ribbon, select **Get data** and choose **Eventstream > New eventstream**.

    ![Screenshot of getting a new eventstream for the Tutorial database.](media/prep-new-eventstream.png)

3. Name your eventstream +++*BusEventstream*+++. When the eventstream is finished creating, it opens.

4. Select **Use sample data**.

    ![Screenshot of selecting sample data for the eventstream.](media/prep-use-sample-data.png)

5. In the **Add source** page, enter +++*BusDataSource*+++ for the source name. Under **Sample data**, select *Buses*. Select **Add**.

    ![Screenshot of selecting the bus sample data.](media/prep-buses.png)

    When the new eventstream is ready, it opens in the authoring canvas.

### Transform data

In this section, you add one transformation to the incoming sample data. This step casts the string fields *ScheduleTime* and *Timestamp* to DateTime type, and renames *Timestamp* to *ArrivalTime* for clarity. Timestamp fields need to be in DateTime format for digital twin builder to use them as time series data.

Follow these steps to add the data transformation. 

1. Select the down arrow on the **Transform events or add destination** tile, then select the **Manage fields** predefined operation. The tile is renamed to *ManageFields*.

    ![Screenshot of selecting the Manage fields operation.](media/prep-manage-fields.png)
    
2. Select the edit icon (shaped like a pencil) on the *MangeFields* tile, which opens the **Manage fields** pane.
3. Select **Add all fields**. This action ensures that all fields from the source data are present through the transformation.
4. Select the *Timestamp* field. Toggle **Change type** to *Yes*. For **Converted Type**, select *DateTime* from the dropdown list. For **Name**, enter the new name of +++*ActualTime*+++.

    ![Screenshot of changing the Timestamp field.](media/prep-manage-fields-2.png)
    
5. Without saving, select the *ScheduleTime* field. Toggle **Change type** to *Yes*. For **Converted Type**, select *DateTime* from the dropdown list. Leave the name as *ScheduleTime*. Now select **Save**.
6. The **Manage fields** pane closes. The **ManageFields** tile continues to display an error until you connect it to a destination.

### Add destination

1. From the menu ribbon, select **Add destination**, then select **Eventhouse**.

    ![Screenshot of selecting the eventhouse destination.](media/prep-add-destination.png)
    
2. Enter the following information in the **Eventhouse** pane:

    | Field | Value |
    | --- | --- |
    | **Data ingestion mode** | Event processing before ingestion |
    | **Destination name** | +++*TutorialDestination*+++ |
    | **Workspace** | Select the workspace in which you created your resources. |
    | **Eventhouse** | *Tutorial* |
    | **KQL Database** | *Tutorial* |
    | **Delta table** | Create new - Enter +++*bus_data_raw*+++ as the table name |
    | **Input data format** | Json |

3. Ensure that the box **Activate ingestion after adding the data source** is checked.
4. Select **Save**.
5. In the authoring canvas, select the **ManageFields** tile and drag the arrow to the **TutorialDestination** tile to connect them. This action resolves all error messages in the flow.
6. From the menu ribbon, select **Publish**. The eventstream now begins sending the sample streaming data to your eventhouse.
7. After a few minutes, the **TutorialDestination** card in the eventstream view displays sample data in the **Data preview** tab. You might need to refresh the preview a few times while you wait for the data to arrive.

    ![Screenshot of the preview data.](media/prep-data-preview.png)

8. Verify that the data table is active in your eventhouse. Go to your *Tutorial* KQL database and refresh the view. It now contains a table called *bus_data_raw* which contains data.

    ![Screenshot of the bus_data_raw table with data.](media/prep-bus-data-raw.png)

## Transform the data using update policies

Now that your bus streaming data is in a KQL database, you can use functions and a [Kusto update policy](/kusto/management/update-policy) to further transform the data. The transformations that you perform in this section prepare the data for use in digital twin builder (preview), and include the following actions:
* Breaking apart the JSON field **Properties** into separate columns for each of its contained data items, **BusStatus** and **TimeToNextStation**. Digital twin builder doesn't have JSON parsing capabilities, so you need to separate these values before the data goes to digital twin builder.
* Adding column **StopCode**, which is a unique key representing each bus stop. The purpose of this step is just to complete the sample data set to support this tutorial scenario. Joinable entities from separate data sources must contain a common column that digital twin builder can use to link them together, so this step adds a simulated set of int values that matches the **Stop_Code** field in the static bus stops data set. In the real world, related data sets already contain some kind of commonality.
* Creating a new table called *bus_data_processed* that contains the transformed data.
* Enabling mirroring for the new table, so that you can use a shortcut to access the data in your *Tutorial* lakehouse. 

Follow these steps to run the queries.
1. From the *Tutorial* KQL database inside your eventhouse, select **Query with code** from the menu ribbon to open the KQL query editor.

    ![Screenshot of the Query with code button on the database.](media/prep-query-with-code.png)

2. Copy and paste the following code into the query editor. Run each code block in order.

    ```kusto
    // Set Columns
    .create-or-alter function extractBusData ()
    {
        bus_data_raw
        | extend BusState = tostring(todynamic(Properties).BusState)
            , TimeToNextStation = tostring(todynamic(Properties).TimeToNextStation)
            , StopCode = toint(10000 + abs(((toint(BusLine) * 100) + toint(StationNumber)) % 750))
        | project-away Properties
    }
    ```

    ```kusto
    // Create table
    .create table bus_data_processed (ActualTime:datetime, TripId:string, BusLine:string, StationNumber:string, ScheduleTime:datetime, BusState:string, TimeToNextStation:string, StopCode:int)
    ```

    ~~~kusto
    // Load data into table
    .alter table bus_data_processed policy update
    ```
    [{
        "IsEnabled": true,
        "Source": "bus_data_raw",
        "Query": "extractBusData",
        "IsTransactional": false,
        "PropagateIngestionProperties": true
    }]
    ```
    ~~~

    ```kusto
    // Enable OneLake availability
    .alter-merge table bus_data_processed policy mirroring dataformat=parquet with (IsEnabled=true, TargetLatencyInMinutes=5)
    ```

    >[!TIP]
    >  You can also enable OneLake availability for the new table through the UI instead of using code. Select the table and toggle on **OneLake availability**.
    >
    > ![Screenshot of enabling OneLake availability in the UI.](media/enable-onelake-availability.png)

3. Optionally, save the query tab as *Bus data processing* so you can identify it later.
4. A new table is created in your database called *bus_data_processed*. After a short wait, it begins to populate with the processed bus data.

    ![Screenshot of the bus_data_processed table with data.](media/prep-bus-data-processed.png)
    
## Create lakehouse shortcut

Finally, create a shortcut that makes the processed bus data available in the *Tutorial* lakehouse, which holds sample data for digital twin builder (preview). This step is necessary because digital twin builder requires its data source to be a lakehouse.

1. Go to your *Tutorial* lakehouse, which you created in the previous tutorial section. From the menu ribbon, select **Get data** > **New shortcut**.

    ![Screenshot of the New shortcut button.](media/prep-new-shortcut.png)

2. Under **Internal sources**, select **Microsoft OneLake**. Then, choose the *Tutorial* KQL database.
3. Expand the list of **Tables** and check the box next to *bus_data_processed*. Select **Next**.
4. Review your shortcut details and select **Create**.

    ![Screenshot of creating the shortcut.](media/prep-new-shortcut-2.png)

5. The *bus_data_processed* table is now available in your lakehouse. The table has a latency of five minutes, so wait five minutes and then verify that it contains data (this might take a few minutes).

    ![Screenshot of the bus_data_processed table in the lakehouse.](media/prep-bus-data-lakehouse.png)

Next, you use this lakehouse data as a source to build an ontology in digital twin builder (preview).

## Next step

> Select **Next >** to Build an ontology in digital twin builder (preview).
