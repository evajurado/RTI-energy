# RTI-energy

1) Let's start by creating in Fabric an Eventstream artifact with a Custom Endpoint Source and a Eventhouse as an output:
<img width="1547" height="329" alt="image" src="https://github.com/user-attachments/assets/df6e306f-fc08-43e7-a9da-f871d0fb9819" />

2) Copy the connection string-primary key from the Custom Endpoint Source, we are going to use it later:
<img width="745" height="515" alt="image" src="https://github.com/user-attachments/assets/4cba1d64-98b9-4a39-909d-b479b59773d7" />

3) In the Eventhouse configuration, use the following details:
<img width="1011" height="939" alt="image" src="https://github.com/user-attachments/assets/7ef32c3a-ac4d-4222-a22c-4ca1d1f23e67" />

4) In the Eventhouse there is a KQL database with a table where all the data is loaded (RTI_Energy_table):
<img width="1910" height="789" alt="image" src="https://github.com/user-attachments/assets/0a5c3b70-d41c-47b3-9416-b1ff28000214" />

5) We are going to create a cleaned version of it, to do that, copy the KQL query stored in the repo in the RTI_Energy_EH_queryset:
<img width="569" height="365" alt="image" src="https://github.com/user-attachments/assets/e66a3dd0-5766-48ca-8576-c15bac6d7557" />

6) Execute the query and you will get a new table RTI_Energy_Stream. 

7) To start the real-time input stream, open the notebook energy_NB.ipynb (stored in this repo) in Fabric and edit in the second cell of the notebook the EVENT_HUB_CONNECTION_STRING value, paste there the connection string-priamry key you have saved in step 2.

8) Execute the whole notebook and it will send fake input data to your Eventstream. If you stop it and you want to relaunch again, you only need to execute the first two cells. 

9) You can go to the Eventhouse or the KQL database to see how the tables fill with new data.

10) Go to RTI_Energy_EH_queryset and create a new tab to generate queries for a new Dashboard. For every query, paste the KQL code and select Save to Dashboard. First time, select "To an new Dashboard" and create the artifact, the following queries, select "To an existing Dashboard".
<img width="1193" height="354" alt="image" src="https://github.com/user-attachments/assets/26903670-9818-400c-b03f-76eaa6e0eec4" />

10.1) Query 1:
    RTI_Energy_Stream
| summarize TotalSales = sum(generated_kwh) by bin(timestamp, 1m)
| where timestamp > ago(1h)
| render timechart 
    with (title="Total Generated kWh Trend", xtitle="Time", ytitle="Generated kWh")

10.2) Query 2:
    let MinTime = toscalar(
    RTI_Energy_Stream
    | where substation_id in ("SUB_007")
    | summarize min(timestamp)
);
let MaxTime = toscalar(
    RTI_Energy_Stream
    | where substation_id in ("SUB_007")
    | summarize max(timestamp)
);
let substations_ts = RTI_Energy_Stream
    | where substation_id in ("SUB_007")
| summarize generated_kwh = sum(generated_kwh) by bin(timestamp, 1m), substation_name;
substations_ts
| partition by substation_name
  (
    // Create the time series array
    make-series Energy_Series = sum(generated_kwh) on timestamp from MinTime to MaxTime step 1m
    | extend (Anomalies, Score, Baseline) = series_decompose_anomalies(Energy_Series, 1.5, -1, 'linefit')
    // Create the highlight series: Sales value only at anomaly points, 0 otherwise.
    | extend AnomalyHighlight = series_multiply(series_abs(Anomalies), Energy_Series)
    | extend Anom = series_abs(Anomalies)
    // Project the results back into a tabular format
    | mv-expand timestamp to typeof(datetime), Energy_Series to typeof(real), AnomalyHighlight to typeof(real), Baseline to typeof(real), Anom to typeof(real)
    | project 
        timestamp, 
        substation_name, 
        Energy=Energy_Series, 
        AnomalyHighlight, 
        Anom
        //Baseline
  )
| where timestamp > ago(1h)
// 4. Render the chart (Simplified to fix syntax error)
| render timechart 
    with (
        title="Energy Trends with Anomalies in Almeria", 
        xtitle="Time", 
        ytitle="Energy Generated (kWh)",
        series=substation_name
    )

10.3) Query 3:
    RTI_Energy_Stream
| where timestamp > ago(1h)
| summarize Generated = sum(generated_kwh) by energy_type
| render piechart 
    with (title="Energy type Generation Trends")

10.4) Query 4:
    RTI_Energy_Stream
| count

10.5) Query 5:
    RTI_Energy_Stream
| summarize 
    Generated = sum(generated_kwh) by substation_id
| join kind=leftouter (external_table('dim_substation')) on substation_id
| project
    Store = substation_name,
    Latitude = latitude,
    Longitude = longitude,
    Generated
| render scatterchart 
    with (kind=map, title="Energy Location")

10.6) Query 6:
    RTI_Energy_Stream
| where timestamp > ago(1h)
| summarize Sales = sum(generated_kwh) by bin(timestamp, 1m) , substation_name
| render timechart 
    with (title="Substation Generation Trends", xtitle="Time", ytitle="(kWh)")

You can add more queries as you wish to configure this dashboard, this is just a suggestion that will allow you to track main parameters. 

Example dashboard output:
<img width="2393" height="1168" alt="image" src="https://github.com/user-attachments/assets/01b97df1-6083-425d-a800-cf9ea6b1c965" />

11) Once the real-time data is ready, we are going to create an ontology to consume the real-time data together with the static data. (The static data was created at the notebook execution from cell three and it is stored in a lakehouse). The ontology is going to have seven entities: Substation, Regulation, Equipment, Maintenance, Weather Station, Technician and Regulation Detail.
The following screenshot shows the semantic model of the data.
<img width="936" height="404" alt="image" src="https://github.com/user-attachments/assets/74f2d1a0-cce7-49b4-bc71-9a845c7a0d7d" />
You have to create first the semantic model, and then the ontology with the "Generate Ontology" button available inside the semantic model artifact.
<img width="264" height="77" alt="image" src="https://github.com/user-attachments/assets/74244155-277e-4f7a-a5e7-60d7c207f4a1" />

12) The real-time data is going to be linked to the Substation entity, for doing that we need to add it as a binding:
<img width="2368" height="1218" alt="image" src="https://github.com/user-attachments/assets/df61c8ea-e77b-47eb-a0b5-c4d2528fc38d" />

13) If you click on Entity type overview, you can see the details of that specific entity:
<img width="242" height="49" alt="image" src="https://github.com/user-attachments/assets/5c9af3de-355c-455a-8777-ea158667234e" />
<img width="2360" height="1113" alt="image" src="https://github.com/user-attachments/assets/76983ae6-90e7-444f-9bc1-1d83b91ff408" />

14) As when creating an ontology there is always a graph underneath, if we open it, we can see the whole model:
<img width="2366" height="1044" alt="image" src="https://github.com/user-attachments/assets/057bcad9-ace0-4ba3-849f-380af6505902" />

15) To query the data using the graphs, we navigate to the left pane and select query and select all the relationships (edges) at the right side under Components:
<img width="2347" height="898" alt="image" src="https://github.com/user-attachments/assets/c558b79d-e52d-4e4b-ab48-954e4f64b5cd" />

16) When running the query, we can see the cloud of data we have in our ontology:
<img width="1186" height="1000" alt="image" src="https://github.com/user-attachments/assets/695b0acd-7e23-4cda-8a73-dd1da23edf14" />

17) By clicking in every node, you can see its properties:
<img width="1512" height="743" alt="image" src="https://github.com/user-attachments/assets/f1fae4ca-13e3-48d1-825a-29fc025f4b0d" />

18) Consuming the ontology data is also possible in a more friendly way in natural language with data agents. We can create one which only data source is the ontology and we can interact with it easily.
<img width="2380" height="1204" alt="image" src="https://github.com/user-attachments/assets/d51e5c30-e4ef-40c6-ae64-16109f0d1500" />

19) To get the best out of the data, we are going to use the anomaly detector artifact in Fabric. Once you create it, you connect it to your data (right pane):
Data: RTI_Energy_Stream (table in KQL database)
Value to watch: generated_kwh
Group_by: substation_name
Timestamp: timestamp
<img width="2360" height="1321" alt="image" src="https://github.com/user-attachments/assets/da034ac5-b808-45ca-95cd-f7359ae4acf7" />
You can customize the detector by selecting the model that best suits your needs and its confidence:
<img width="818" height="894" alt="image" src="https://github.com/user-attachments/assets/4150d392-a0aa-4834-a95e-d1d5afb0b73c" />
You can also filter and choose the instances (substations) you want to watch, no need to visualize everything:
<img width="2341" height="1194" alt="image" src="https://github.com/user-attachments/assets/c3b0ae19-c78a-47ad-9d16-aa4a620b67c9" />

20) Bonus check: let's finish with a Map that locates the subtations, the repairing offices and how the technician's trucks move from the repairing offices to the substations:
Substations and repairing offices:
<img width="2075" height="1124" alt="image" src="https://github.com/user-attachments/assets/341c8638-51b1-4662-a641-5b4c5d218701" />
+ trucks data:
<img width="2152" height="1148" alt="image" src="https://github.com/user-attachments/assets/d4a10130-9818-4556-b750-bbba03ae60b5" />

To generate the trucks' data, there is an additional notebook in the repo: Trucks_NB. It follows the same RTI logic to generate the content. 




