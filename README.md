# Time Series Demo 
Time-series demo involving BigQuery and BigTable

## Download the demo data and table definition file: 
In the folder demo-data you will find two files:
- **bigtable_import.csv** time-series demo data, already formatted for BigTable
- **raw.csv** same data as above, just human-friendly formatted and data ready to be uploaded to BigQuery (not mandatory)

In the main folder, you will find one file:
- **tabledef.json** table defitinion file


## Preparation steps
1. Create BigTable cluster (default timeseries-bt)
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
cbt createtable timeseries-bt families="cell_data"
```
5. Import data into BigTable
```
cbt import timeseries-bt bigtable_import.csv column-family=cell_data
```
6. Verify the dataload is ok
```
cbt read timeseries-bt
```

## Step 2: Create a federated table in BigQuery
1. Verify the BigTable URI contained in the table definition file tabledef.json; format is      `https://googleapis.com/bigtable/projects/<projectid>/instances/<instanceid>/tables/<tableid>`
```
{
    "sourceFormat": "BIGTABLE",
    "sourceUris": [
     https://googleapis.com/bigtable/projects/db-timeseries/instances/timeseries/tables/timeseries-bt
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
FROM `db-timeseries-sky.timeseries.btfederated` LIMIT 10

```
