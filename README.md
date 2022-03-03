# Time Series Demo 
Time-series demo involving BigQuery and BigTable

## Download the demo data
Clone this repository, folder demo-data. Two files are in stored there:
- **bigtable_import.csv** time-series demo data, already formatted for BigTable
- **raw.csv** same data as above, just human-friendly formatted and data ready to be uploaded to BigQuery (not mandatory)

## Preparation steps
1. Create BigTable cluster (default timeseries-bt)
2. Create BigQuery dataset (default db-timeseries:timeseries)
3. Upload the demo data (bigtable_import.csv) locally to Cloud Shell

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
