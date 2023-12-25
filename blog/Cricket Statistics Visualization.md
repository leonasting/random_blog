This blog is about building simple ETL pipeline for data visualization. The content present in the blog based on Youtube video with title "[Cricket Statistics Data Pipeline in Google Cloud using Airflow | Data Engineering Project](https://www.youtube.com/watch?v=UXJxcWgxwu0)". 

**Tech Stack**
We have used the below tech stack for this project
* Language - Python ,Big Query 
* Data Orchestration - Airflow , Cloud Data Flow, Google Cloud Functions
* Storage - Google Cloud Storage 
* Dashboard - Looker Studio 

**Prerequisite**

* RapidAPI Account  - To call CricBuzz API
	* Please create account and subscribe the basic plan of no cost.
* Google Cloud Account - To have storage and BigQuery platform
	* Python Package google cloud storage
		`pip install google-cloud-storage
`

Following are the steps that we performed setup out pipeline:
1.  [Fetching statistic data from source and loading it in google cloud storage.](## Step 1 Fetching statistic data from source and loading it in google cloud storage.)


## Step1: Create Bucket a in Google Cloud Storage.

Create a new Project or use the default project.
Go to [Cloud Storage](https://console.cloud.google.com/storage/).
Create a Bucket with Name "bkt-ranking-data"
Ensure that the google application credentials is present in the form of json.
Note: if it is not present use the this link [Create and delete service account key](https://cloud.google.com/iam/docs/keys-create-delete)

### Creation of Service Account

Provide Information as presented in the fig. below
![[Service Account Creation.png]]

Once the service account is created. Create the key using Action button -> Mange Keys as presented in below figure.
![[Manage Key.png]]
Click on Add Key-> Create new key. It will open a pop up. Select JSON file type.

![[JSONKey Creation.png]]


## Step 1 Fetching statistic data from source and loading it in google cloud storage.

We will be using python to extract the data using Rapid API and store it in local file system.
The stored data will be uploaded the cloud bucket.
Below is the code snippet used to perform the above task.

``` python
# storage key file provided by GCP used to authenticate the user to access the GCS bucket
storage_client = storage.Client.from_service_account_json(keys["gcp_key_file"])
def upload_to_gcs(csv_filename,bucket_name = 'bkt-ranking-data'):
    # Upload the CSV file to GCS
    bucket = storage_client.bucket(bucket_name)
    destination_blob_name = f'{csv_filename}'  # The path to store in GCS

    blob = bucket.blob(destination_blob_name)# blob object to store the file in GCS
    blob.upload_from_filename(csv_filename)

    print(f"File {csv_filename} uploaded to GCS bucket {bucket_name} as {destination_blob_name}")


def fetch_data_from_api():
    response = requests.get(url, headers=headers, params=params)
    if response.status_code == 200:
        data = response.json().get('rank', [])  # Extracting the 'rank' data
        csv_filename = 'batsmen_rankings.csv'

        if data:
            field_names = ['rank', 'name', 'country']  # Specify required field names

            # Write data to CSV file with only specified field names
            with open(csv_filename, 'w', newline='', encoding='utf-8') as csvfile:
                writer = csv.DictWriter(csvfile, fieldnames=field_names)
                # writer.writeheader()
                for entry in data:
                    writer.writerow({field: entry.get(field) for field in field_names})

            print(f"Data fetched successfully and written to '{csv_filename}'")
            upload_to_gcs(csv_filename)
        else:
            print("No data available from the API.")
    else:
        print("Failed to fetch data:", response.status_code)

```

## Step 2  Creating Data Flow Job to transfer data from cloud storage to bigQuery

Open the [dataflow](https://console.cloud.google.com/dataflow/) in the console.
Click on the button "Create job from the template" as presented in the image below.

![[create job in dataflow.png]]
(Mandatory) Enable the the Dataflow API
![[Data Flow API image.png]]
Create Dataflow using template.
Following things were selected for batch creation:
* Name: "XXXX" - Any Unique Identifier
* Region: "Region Close to you location"
* Template: `Process Data in Bulk(batch)/Text File on Cloud Storage to BigQuery
`
![[create job.png]]

It will ask for Required Parameters.

* JavaScript User Defined Function - Get the url from object details: gs://bkt-dataflow-metadata/udf.js
* JSON files containing BigQuery Schema File. - It can be browsed.
* Function Name is present in the udf fike - transform
* BigQuery Output table - select the table created in BigQuery
* Temporary BigQuery directory - Use the metadata bucket :"bkt-dataflow-metadata"
* Temporary BigQuery folder location -  Use the metadata bucket :"bkt-dataflow-metadata/temp


We can observe the Job Execution in the form of graph view.
![[dataflow graph.png]]


We can check in BigQuery to view contents of the data.


**JSON File**
BigQuery Schema file is present in the repository contains the following code defining the structure of table headers. 
* rank   -  String : Player's  current Rank
* name -  String : Player's Name
* country - String : Player's Country 
```
{
    "BigQuery Schema": [{
      "name": "rank",
      "type": "STRING"
     },
     {
      "name": "name",
      "type": "STRING"
     },
     {
      "name": "country",
      "type": "STRING"
     }
    ]
   }
```

**JavaScript File**
The code present in the User Defined Function is present below. It act as mapping function to transform raw data from cloud storage to BigQuery objects.
```python
function transform(line) {
    var values = line.split(',');
    var obj = new Object();
    obj.rank = values[0];
    obj.name = values[1];
    obj.country = values[2];
    var jsonString = JSON.stringify(obj);
    return jsonString;
   }
```


### Setup Table in BigQuery
Create dataset in BigQuery with dataset name - cricket db. The following image shows the button to create dataset.
![[dataset bigquery.png]]

Create Table "icc_odi_batsman_ranking" by using three dot next to cricket_db
![[create table.png]]

Note: Schema details will be provided by the JSON file.

## Setup Bucket to Store the Meta data of DataFlow

Create Bucket with name "bkt-dataflow-metadata".
Upload files following files from repository to the bucket.
* bq.json : JSON file containing schema for BigQuery Database
* udf.js : JS file contaning mapping function

Capture the path of individual file that will be required for dataflow job creation.
![[object path.png]]

Other than uploading files create temporary folder "temp". It will used as temporary location for DataFlow job execution.
