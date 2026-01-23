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
7) To start the real-time input stream, open the notebook RTI_Energy_NB.ipynb (stored in this repo) in Fabric and edit in the second cell of the notebook the EVENT_HUB_CONNECTION_STRING value, paste there the connection string-priamry key you have saved in step 2.
8) Execute the whole notebook and it will send fake input data to your Eventstream.
9) You can go to the Eventhouse or the KQL database to see how the tables fill with new data.
10) Go to RTI_Energy_EH_queryset and create a new tab to generate queries for a new Dashboard. For every query, paste the KQL code and select Save to Dashboard. First time, select "To an new Dashboard" and create the artifact, the following queries, select "To an existing Dashboard".
<img width="1193" height="354" alt="image" src="https://github.com/user-attachments/assets/26903670-9818-400c-b03f-76eaa6e0eec4" />
10.1) Query 1:
    RTI_Energy_Stream
| summarize TotalSales = sum(generated_kwh) by bin(timestamp, 1m)
| where timestamp > ago(1h)
| render timechart 
    with (title="Total Generated kWh Trend", xtitle="Time", ytitle="Generated kWh")
10.2 Query 2:
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

YOu can add more queries as you wish to configure this dashboard, this is just a suggestion that will allow you to track main parameters. 

   
