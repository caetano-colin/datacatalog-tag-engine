## Tag Engine 2.0
This is the main branch for Tag Engine. Tag Engine v2 is a flavor of Tag Engine that is hosted on Cloud Run instead of App Engine and is [VPC-SC compatible](https://cloud.google.com/vpc-service-controls/docs/supported-products). It supports user authentication and role based access control. Customers who have multiple teams using BigQuery and Cloud Storage can authorize each team to tag only their data assets using separate Tag Creator service accounts (more on that later). 

Tag Engine is an open-source extension to Google Cloud's Data Catalog which is now part of the Dataplex product suite. Tag Engine automates the tagging of BigQuery tables and views as well as data lake files in Cloud Storage. You create tag configurations that specify how to populate the various fields of a tag template through SQL expressions or static values. Tag Engine runs the configurations either on demand or on a schedule to create, update or delete the tags.

This README file contains deployment steps, testing procedures, and code samples. It is organized into five sections:  <br>
- Part 1: [Deploying Tag Engine v2](#deploy) <br>
- Part 2: [Testing your Tag Engine API Setup](#testa)  <br>
- Part 3: [Testing your Tag Engine UI Setup](#testb)  <br>
- Part 4: [Troubleshooting](#troubleshooting)  <br>
- Part 5: [Next Steps](#next)  <br> 

### <a name="deploy"></a> Part 1: Deploying Tag Engine v2

Tag Engine 2.0 comes with two Cloud Run services. One service is for the API (`tag-engine-api`) and the other is for the UI (`tag-engine-ui`). 

Both services use access tokens for authorization. The API service expects the client to pass in an access token when calling the API functions (`gcloud auth print-identity-token`) whereas the UI service uses OAuth to authorize the client from the front-end. Note that a client secret file is required for the OAuth flow.  

Follow the steps below to deploy Tag Engine with Terraform. 

Alternatively, you may choose to deploy Tag Engine with [gcloud commands](https://github.com/GoogleCloudPlatform/datacatalog-tag-engine/tree/cloud-run/docs/manual_deployment.md) instead of running the Terraform.

1. Create (or designate) two service accounts: <br>
   - A service account that runs the Tag Engine Cloud Run services (both API and UI). This account is referred to as `TAG_ENGINE_SA`. 
   - A service account that sources the metadata from BigQuery or Cloud Storage, and then performs the tagging in Data Catalog. This account is referred to as `TAG_CREATOR_SA`. <br>

   See [Creating Service Accounts](https://cloud.google.com/iam/docs/service-accounts-create) for more details.

   Why do we need two different service accounts? The key benefit of decoupling them is to allow individual teams to have their own Tag Creator SA. This account has permissions to read specific data assets in BigQuery and Cloud Storage. For example, the Finance team can have a different Tag Creator SA from the Finance team if they own different data assets. The Tag Engine admin then links each invoker account (either service or user) to a specific Tag Creator SA. Invoker accounts call Tag Engine through either the API or UI. This allows the Tag Engine admin to run and maintain a single instance of Tag Engine, as opposed to one instance per team. <br>

2. Create an OAuth client:

   Open [API Credentials](https://console.cloud.google.com/apis/credentials).<br>

   Click on Create Credentials and select OAuth client ID and choose the following settings:<br>

   Application type: web application<br>
   Name: tag-engine-oauth<br>
   Authorized redirects URIs: <i>Leave this field blank for now.</i>  
   Click Create<br>
   Download the credentials as `te_client_secret.json` and place the file in the root of the `datacatalog-tag-engine` directory<br>

   Note: The client secret file is required for establishing the authorization flow from the UI.  

3. Make a copy of `datacatalog-tag-engine/tagengine.ini.tpl` naming the new copy `datacatalog-tag-engine/tagengine.ini`.

4. Open `datacatalog-tag-engine/tagengine.ini` and set the following variables in this file: 
	```
	TAG_ENGINE_SA
	TAG_CREATOR_SA
	TAG_ENGINE_PROJECT
	TAG_ENGINE_REGION
	FIRESTORE_PROJECT
	FIRESTORE_REGION
	FIRESTORE_DATABASE 
	BIGQUERY_REGION
	FILESET_REGION
	SPANNER_REGION
	ENABLE_AUTH
	OAUTH_CLIENT_CREDENTIALS
	ENABLE_TAG_HISTORY
	TAG_HISTORY_PROJECT
	TAG_HISTORY_DATASET
	ENABLE_JOB_METADATA
	JOB_METADATA_PROJECT
	JOB_METADATA_DATASET  
	```

   A couple of notes: <br>

   - The variable `ENABLE_AUTH` is a boolean. When set to `True`, Tag Engine verifies that the end user is authorized to use `TAG_CREATOR_SA` prior to processing their tag requests. This is the recommended value. <br>

   - The `tagengine.ini` file also has two additional variables, `INJECTOR_QUEUE` and `WORK_QUEUE`. These determine the names of the cloud task queues. You do not need to change them. If you change their name, you need to also change them in the `deploy/variables.tf`.<br><br> 

5. Create a new GCS bucket for CSV imports. Remember GCS bucket names are globally unique. 
	For example: `gsutil mb gs://$(gcloud config get-value project)-csv-import`

6. Set the Terraform variables:

   Open `deploy/variables.tf` and change the default value of each variable.<br>
   Save the file.<br><br> 
   Alternatively, create a new file, named `deploy/terrform.tfvars` and specify your variables values there.  


7. Run the Terraform scripts:
	
	__NOTE__: The terraform script will run with the default credentials currently configured on your system. Make sure that your current user has the required permissions to make changes to your project(s), or set new credentials using the `GOOGLE APPLICATION_CREDENTIALS` environment variable.
	
	```
	cd deploy
	terraform init
	terraform plan
	terraform apply
	```

	When the Terraform finishes running, it should output two URIs. One for the API service (which looks like this https://tag-engine-api-xxxxxxxxxxxxx.a.run.app) and another for the UI service (which looks like this https://tag-engine-ui-xxxxxxxxxxxxx.a.run.app). <br><br>

### <a name="testa"></a> Part 2: Testing your Tag Engine API setup

1. Create the sample `data_governance` tag template:

	```
	git clone https://github.com/GoogleCloudPlatform/datacatalog-templates.git 
	cd datacatalog-templates
	python create_template.py $DATA_CATALOG_PROJECT $DATA_CATALOG_REGION data_governance.yaml 
	```

	The previous command creates the `data_governance` tag template in the `$DATA_CATALOG_PROJECT` and `$DATA_CATALOG_REGION`. 

2. Grant permissions to invoker account (user or service)

	Depending on how you are involving the Tag Engine API, you'll need to grant permissions to either your service account or user account (or both). 

	If you'll be invoking the Tag Engine API with a user account, authorize your user account as follows:

	```
	gcloud auth login
	
	export INVOKER_USER_ACCOUNT="username@example.com"

	gcloud iam service-accounts add-iam-policy-binding $TAG_CREATOR_SA \
	--member=user:$INVOKER_USER_ACCOUNT --role=roles/iam.serviceAccountUser --project=$DATA_CATALOG_PROJECT

	gcloud iam service-accounts add-iam-policy-binding $TAG_CREATOR_SA \
	--member=user:$INVOKER_USER_ACCOUNT --role=roles/iam.serviceAccountTokenCreator --project=$DATA_CATALOG_PROJECT 

	gcloud run services add-iam-policy-binding tag-engine-api \
	--member=user:$INVOKER_USER_ACCOUNT --role=roles/run.invoker \
	--project=$TAG_ENGINE_PROJECT --region=$TAG_ENGINE_REGION 
	```

	If you are invoking the Tag Engine API with a service account, authorize your service account as follows:

	```
	export INVOKER_SERVICE_ACCOUNT="tag-engine-invoker@<PROJECT>.iam.gserviceaccount.com"

	gcloud iam service-accounts add-iam-policy-binding $TAG_CREATOR_SA \
		--member=serviceAccount:$INVOKER_SERVICE_ACCOUNT --role=roles/iam.serviceAccountUser 

	gcloud iam service-accounts add-iam-policy-binding $TAG_CREATOR_SA \
		--member=serviceAccount:$INVOKER_SERVICE_ACCOUNT --role=roles/iam.serviceAccountTokenCreator 

	gcloud run services add-iam-policy-binding tag-engine-api \
		--member=serviceAccount:$INVOKER_SERVICE_ACCOUNT --role=roles/run.invoker \
		--region=$TAG_ENGINE_REGION	
	```

	<b>Very important: Tag Engine requires that these roles be directly attached to your invoker account(s).</b> 

3. Generate an IAM token (aka Bearer token) for authenticating to Tag Engine: 

	If you are invoking Tag Engine with a user account, run `gcloud auth login` and authenticate with your user account. 
	If you are invoking Tag Engine with a service account, set `GOOGLE_APPLICATION_CREDENTIALS`. 

	```
	export IAM_TOKEN=$(gcloud auth print-identity-token)
	```

4. Create your first Tag Engine configuration:

	Tag Engine uses configurations (configs for short) to define tag requests. There are several types of configs from ones that create dynamic table-level tags to ones that create tags from CSV. You'll find several example configs in the `examples/configs/` subfolders.

	For now, open `examples/configs/dynamic_table/dynamic_table_ondemand.json` and update the project and dataset values in this file to match your Tag Engine and BigQuery environments.  

	```
    cd <PATH_TO_TAG_ENGINE_PROJECT>
	export TAG_ENGINE_URL=$SERVICE_URL

	curl -X POST $TAG_ENGINE_URL/create_dynamic_table_config -d @examples/configs/dynamic_table/dynamic_table_ondemand.json \
		 -H "Authorization: Bearer $IAM_TOKEN"
	```

	Note: `$SERVICE_URL` should be equal to your Cloud Run URL for `tag-engine-api`. 

	The output from the previous command should look similar to:

	```
	{"config_type":"DYNAMIC_TAG_TABLE","config_uuid":"facb59187f1711eebe2b4f918967d564"}
	```

5. Run your first job:

	Now that we have created a config, we need to trigger it in order to create the tags. A Tag Engine job is an execution of a config. In this step, you execute the dynamic table config using the config_uuid from the previous step. 

	Note: Before running the next command, please update the `config_uuid` with your own value. 

	```
	curl -i -X POST $TAG_ENGINE_URL/trigger_job \
		-d '{"config_type":"DYNAMIC_TAG_TABLE","config_uuid":"facb59187f1711eebe2b4f918967d564"}' \
		-H "Authorization: Bearer $IAM_TOKEN"
	```

	The output from the previous command should look similar to:

	```
	{
		"job_uuid": "069a312e7f1811ee87244f918967d564"
	}
	```

	If you enabled job metadata in `tagengine.ini`, you can optionally pass a job metadata object to the trigger_job call. This gets stored in BigQuery, along with the job execution details. Please note that the job metadata option is not required, you can skip this step:
	
	```
	curl -i -X POST $TAG_ENGINE_URL/trigger_job \
		-d '{"config_type":"DYNAMIC_TAG_TABLE","config_uuid":"c255f764d56711edb96eb170f969c0af","job_metadata": {"source": "Collibra", 		"workflow": "process_sensitive_data"}}' \
		-H "Authorization: Bearer $IAM_TOKEN"
	```
	
	The job metadata parameter gets written into a BigQuery table that is associated with the job_uuid. 


6. View your job status:

	Note: Before running the next command, please update the `job_uuid` with your value. 

	```
	curl -X POST $TAG_ENGINE_URL/get_job_status -d '{"job_uuid":"069a312e7f1811ee87244f918967d564"}' \
		-H "Authorization: Bearer $IAM_TOKEN"
	```

	The output from this command should look like this:

	```
		{
	  	  "job_status": "SUCCESS",
	  	  "task_count": 1,
	  	  "tasks_completed": 1,
	  	  "tasks_failed": 0,
	  	  "tasks_ran": 1
	}
	```
	
	Open the Data Catalog UI and verify that your tag was successfully created. If your tag is not there or if you encounter an error with the previous commands, open the Cloud Run logs for the `tag-engine-api` service and investigate. 

<br>

### <a name="testb"></a> Part 3: Testing your Tag Engine UI Setup

1. Set the authorized redirect URI and add authorized users:

    - Re-open [API Credentials](https://console.cloud.google.com/apis/credentials)<br>

    - Under OAuth 2.0 Client IDs, edit the `tag-engine-oauth` entry which you created earlier. <br>

    - Under Authorized redirect URIs, add the URI:
      https://tag-engine-ui-xxxxxxxxxxxxx.a.run.app/oauth2callback
	
    - Replace xxxxxxxxxxxxx in the URI with the actual value from the Terraform. This URI will be referred to below as the `UI_SERVICE_URI`.

    - Open the OAuth consent screen page and under the Test users section, click on add users.

    - Add the email address of each user for which you would like to grant access to the Tag Engine UI. 

2. Grant permissions to your invoker user account(s):

    ```
    export INVOKER_USER_ACCOUNT="username@example.com"`

    gcloud iam service-accounts add-iam-policy-binding $TAG_CREATOR_SA \
        --member=user:$INVOKER_USER_ACCOUNT --role=roles/iam.serviceAccountUser
    ```

3. Open a browser window
4. Navigate to `UI_SERVICE_URI` 
5. You should be prompted to sign in with Google
6. Once signed in, you will be redirected to the Tag Engine home page (i.e. `UI_SERVICE_URI`/home)
7. Enter your template id, template project, and template region
8. Enter your `TAG_CREATOR_SA` as the service account
9. Click on `Search Tag Templates` to continue to the next step 
10. Create a tag configuration by selecting one of the options from this page. 

     If you encounter a 500 error, open the Cloud Run logs for `tag-engine-ui` to troubleshoot. 
<br>

### <a name="troubleshooting"></a> Part 4: Troubleshooting

There is a known issue with the Terraform. If you encounter the error `The requested URL was not found on this server` when you try to create a configuration from the API, the issue is that the container didn't build correctly. Try to rebuild and redeploy the Cloud Run API service with this command:

```
    cd datacatalog-tag-engine
    gcloud run deploy tag-engine-api \
 	--source . \
 	--platform managed \
 	--region $TAG_ENGINE_REGION \
 	--no-allow-unauthenticated \
 	--ingress=all \
 	--memory=4G \
	--timeout=60m \
 	--service-account=$TAG_ENGINE_SA
```

Then, call the `ping` endpoint as follows:

```
    curl $TAG_ENGINE_URL/ping -H "Authorization: Bearer $IAM_TOKEN"
```

You should see the following response:

```
    Tag Engine is alive
```

### <a name="next"></a> Part 5: Next Steps

1. Explore additional API methods and run them through curl commands:

   Open `examples/unit_test.sh` and go through the different methods for interracting with Tag Engine, including `configure_tag_history`, `create_static_asset_config`, `create_dynamic_column_config`, etc. <br>

2. Explore the script samples:

   There are multiple test scripts in Python in the `examples/scripts` folder. These are intended to help you get started with the Tag Engine API. 

   Before running the scripts, open each file and update the `TAG_ENGINE_URL` variable on line 11 with your own Cloud Run service URL. You'll also need to update the project and dataset values which may be in the script itself or in the referenced json config file. 

   Here are some of the scripts you can look at and run:

	```
	python configure_tag_history.py
	python create_static_config_trigger_job.py
	python create_dynamic_table_config_trigger_job.py
	python create_dynamic_column_config_trigger_job
	python create_dynamic_dataset_config_trigger_job.py
	python create_import_config_trigger_job.py
	python create_export_config_trigger_job.py
	python list_configs.py
	python read_config.py
	python purge_inactive_configs.py
	```

3. Explore sample workflows:

   The `extensions/orchestration/` folder contains some sample workflows implemented in Cloud Workflow. The `trigger_tag_export.yaml` and `trigger_tag_export_import.yaml` show how to orchestrate Tag Engine jobs. To run the workflows, enable the Cloud Workflows API (`workflows.googleapis.com`) and then follow these steps:

	```
	gcloud workflows deploy orchestrate-jobs --location=$TAG_ENGINE_REGION \
		--source=trigger_export_import.yaml --service-account=$CLOUD_RUN_SA

	gcloud workflows run trigger_export_import --location=$TAG_ENGINE_REGION
	```
	In addition to the Cloud Workflow examples, there are two examples for Airflow in the same folder, `dynamic_tag_update.py` and `pii_classification_dag.py`.  

<br>

4. Create your own Tag Engine configs with the API and/or UI. <br>

<br>

5. Open new [issues](https://github.com/GoogleCloudPlatform/datacatalog-tag-engine/issues) if you encounter bugs or would like to request a new feature or extension. 
