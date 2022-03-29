# Running Analytics for Time Series in GCP: crash demo

Crash demo to show:
- how to Federate a BigTable table containing time series data to BigQuery
- how to run BiqQuery Analytics query on the federated table
- how to transform the federated table into a native table, and harness the analytical power of BigQuery

## Download the demo data and table definition file: 
In the folder demo-data you will find two files:
- **bigtable_import.csv** time-series demo data, already formatted for BigTable
- **raw.csv** same data as above, just human-friendly formatted and data ready to be uploaded to BigQuery (not mandatory)

In the main folder, you will find one file:
- **tabledef.json** table defitinion file


## Preparation steps
1. Create BigTable cluster (default timeseries)
2. Create BigQuery dataset (default db-timeseries:timeseries)
3. Upload the demo data (bigtable_import.csv) and the table definition file (tabledef.json) locally to Cloud Shell

## Step 1: Load data into BigTable
Reference: [Easy CSV importing into cloud Bigtable](https://medium.com/google-cloud/easy-csv-importing-into-cloud-bigtable-ed3f62139b89) 
1. Open Cloud Shell
2. List the Bigtable instances
```
cbt listinstances
```
3. Prepare the cbtrc file:
```
echo project = db-timeseries > ~/.cbtrc
echo instance = timeseries >> ~/.cbtrc
```
4. Create BigTable table and column family
```
cbt createtable timeseries families="cell_data"
```
5. Import data into BigTable
```
cbt import timeseries bigtable_import.csv column-family=cell_data
```
6. Verify the dataload is ok
```
cbt read timeseries
```

## Step 2: Create a federated table in BigQuery
1. Verify the BigTable URI contained in the table definition file tabledef.json; format is      `https://googleapis.com/bigtable/projects/<projectid>/instances/<instanceid>/tables/<tableid>`
```
{
    "sourceFormat": "BIGTABLE",
    "sourceUris": [
     https://googleapis.com/bigtable/projects/openreachday2022/instances/bigtable-timeseries/tables/bigtable-timeseries
    ],
    "bigtableOptions": {
        "readRowkeyAsString": "true",
        "columnFamilies" : [
            {
                "familyId": "cell_data",
                "onlyReadLatest": "true",
                "columns": [
              {
                  "qualifierString": "attr1",
                  "type": "STRING"
              },
              {
                  "qualifierString": "attr2",
                  "type": "STRING"
              }
              ,
                  {
                  "qualifierString": "attr3",
                  "type": "STRING"
              }
              ,
                  {
                  "qualifierString": "attr4",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr5",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr6",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr7",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr8",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr9",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "attr10",
                  "type": "STRING"
              }
              ,
              {
                  "qualifierString": "tstamp",
                  "type": "STRING"
              }
          ]

            }
        ]
    }
    
}

```
Note: this table definition file will only federate three columns: `attr1`, `attr2`, `tstamp`

2. In Cloud Shell, create a permanent BigQuery external table:
```
bq mk \
--external_table_definition=tabledef.json \
timeseries.btfederated
```
3. In the BigQuery Web UI, run a query on the newly federated table: 
```
SELECT rowkey, 
cell_data.attr1.cell.value as attr1_raw, 
cell_data.attr2.cell.value as attr2_raw, 
cell_data.tstamp.cell.value as ts_raw, 


cast(cell_data.attr1.cell.value as NUMERIC) attr1,
cast(cell_data.attr2.cell.value as NUMERIC) attr2,
timestamp(cell_data.tstamp.cell.value) ts
FROM `db-timeseries.timeseries.btfederated` LIMIT 10

```

## Step 3: Run Analytics on the federated table
1. Create a BigQuery view on the newly imported data:
```
CREATE VIEW `db-timeseries.timeseries.view` AS
SELECT timestamp(cell_data.tstamp.cell.value) ts,
cast(cell_data.attr1.cell.value as NUMERIC) attr1,
cast(cell_data.attr2.cell.value as NUMERIC) attr2
 FROM `db-timeseries.timeseries.btfederated` 

```
2. Run an aggregation on that view:
```
SELECT extract(hour from ts) as hour, sum(attr1) as sum_attr1, sum(attr2)  as sum_attr2 FROM `db-timeseries.timeseries.view` 
group by extract(hour from ts)
```

Appreciate that performance is still suboptimal since we’re using the BigTable execution engine. 
We need to convert the underlying table `db-timeseries.timeseries.btfederated` from **external** to **native**.


## Step 4: Convert to native table, re-run analytics
Reference: [Is it possible to convert external tables to native in BigQuery?](https://stackoverflow.com/questions/43386615/is-it-possible-to-convert-external-tables-to-native-in-bigquery)

1. Create an empty BigTable destination table, default name `db-timeseries.timeseries.native`

2. Run the following query [making sure to store its result to the newly created permanent destination table](https://cloud.google.com/bigquery/docs/writing-results):
```
SELECT  ts, attr1, attr2
FROM `db-timeseries-sky.timeseries.view`
```
Suggest to set the **Destination table write preference** property to **Overwrite table**

3. Run the basic and aggregation queries we ran before on the newly created permanent table, to appreciate the difference in speed: 

```
SELECT  ts, attr1, attr2
FROM `db-timeseries-sky.timeseries.native`
```

```
SELECT extract(hour from ts) as hour, sum(attr1) as sum_attr1, sum(attr2)  as sum_attr2 FROM `db-timeseries.timeseries.native` 
group by extract(hour from ts)
```
