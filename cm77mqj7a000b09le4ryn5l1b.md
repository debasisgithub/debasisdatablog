---
title: "Unity Catalog Integration With Microsoft Fabric"
datePublished: Sun Feb 16 2025 12:55:39 GMT+0000 (Coordinated Universal Time)
cuid: cm77mqj7a000b09le4ryn5l1b
slug: unity-catalog-integration-with-microsoft-fabric
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/Ype9sdOPdYc/upload/a3823f58958e2074b93e4899b883dac4.jpeg
tags: authentication, databricks, microsoftfabric, unity-catalog, fabricrestapi

---

In the catalog service offerings Unity Catalog (created by Databricks later open sourced) is unquestionably superior and has established itself as a leading solution in the market. And its integration with many other products and services are growing.

However currently as I am writing this blog there is no market ready default integration available with Microsoft Fabric which is anyway a new product.

I will be explaining and showcasing how we can integrate the metadata from unity catalog to the Fabric Lakehouse and create the same schemas and tables as shortcuts.

### What You Need

You need to have a ready fabric workspace attached to a running capacity, a fabric Lakehouse inside your workspace, fabric cloud connection to the external storage account(e.g. ADLS Gen2) and last but not the least an existing unity catalog and schemas in your databricks workspace.

I consider you already have everything ready in your environment however I will explain bit details about the cloud connection you need to create for this. We need to create this cloud storage connection to the same ADLS gen2 storage which your unity catalog tables are using.

This is how you can create a new connection. You can choose between multiple authentication method.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1739696844081/4954b929-b223-41fe-9756-301749accccc.png align="left")

Once you create the cloud connection you can fetch your connection id (see below example) from **Settings**\&gt; **Manage connections and gateways** &gt; **Connections** &gt; **Settings**

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1739697283322/4b7c4664-88f2-49aa-abad-1851c5d177d1.png align="left")

### How it works

You can create a notebook(see attached github link) in your fabric workspace and try execute the below code blocks, you can do similar operation from databricks workspace as well, provided you have necessary privilege meaning you need to add the same Service Principal to the databricks Workspace.

[Databricks Unity Catalog Integration Notebook](https://github.com/debasisgithub/dataengineering_AI/blob/main/Databricks_UC_Integration/dbr_unity_catalog_integration.py)

![Screenshot showing Unity Catalog to Fabric shortcuts flow.](https://learn.microsoft.com/en-us/fabric/onelake/media/onelake-unity-catalog/unity-catalog-fabric-flow.png align="left")

1. We are using unity catalog REST APIs to extracts the metadata information from the given catalog and schema.
    
2. The REST API get request will return the external table information details from databricks unity catalog, you need make the request using Databricks Personal Access Token(PAT) or OAuth. Here we are using PAT. The below code snippet will return the unity catalog schema and table details.
    
    ```python
    def get_databricks_uc_tables(databricks_config):    
        all_tables = []
    
        dbx_workspace = databricks_config['dbx_workspace']
        dbx_token = databricks_config['dbx_token']
        dbx_uc_catalog = databricks_config['dbx_uc_catalog']
        dbx_uc_schemas = databricks_config['dbx_uc_schemas']
    
        for schema in dbx_uc_schemas:
            url = f"{dbx_workspace}/api/2.1/unity-catalog/tables?catalog_name={dbx_uc_catalog}&schema_name={schema}"
            payload = {}
            headers = {
                'Authorization': f'Bearer {dbx_token}',
                'Content-Type': 'application/json'
            }
            response = requests.get(url, data=payload, headers=headers)
    
            if response.status_code == 200:
                response_json = json.loads(response.text)
                all_tables.extend(response_json['tables'])
            else:
                print(f"Error Occurred: [{response.status_code}] Cannot connect to Unity Catalog. Please review configs.")
                return None
        return all_tables
    ```
    
    3. Once you get the details you can create the fabric shortcuts using the fabric REST api. Currently as per Microsoft documenation for shortcut creation using REST APIs only user identity authentication is supported other authentication method as Service Principal, Workspace Identity is not supported now. The below methods will create the shortcuts to Fabric Lakehouse. You can generate the user identity authentication token separately using POSTMAN or you can try it here
        
        [https://learn.microsoft.com/en-us/rest/api/fabric/core/onelake-shortcuts/create-shortcut?tabs=HTTP#code-try-0](https://learn.microsoft.com/en-us/rest/api/fabric/core/onelake-shortcuts/create-shortcut?tabs=HTTP#code-try-0)
        
        and sign in with your user AD account then you will have the bearer token in the request preview. You can use the token here in the automation script.
        
        ```python
        FABRIC_API_ENDPOINT = "api.fabric.microsoft.com"
        ONELAKE_API_ENDPOINT = "onelake.dfs.fabric.microsoft.com"
        
        def get_lakehouse_shortcuts(fabric_config):
            workspace_id = fabric_config['workspace_id']
            lakehouse_id = fabric_config['lakehouse_id']
        
            table_infos = dbutils.fs.ls(f"abfss://{workspace_id}@{ONELAKE_API_ENDPOINT}/{lakehouse_id}/Tables")
            current_shortcut_names = [table_info.name for table_info in table_infos]
            return current_shortcut_names
        
        def create_shortcuts(fabric_config, tables):    
            sc_created, sc_failed, sc_skipped = 0, 0, 0
        
            workspace_id = fabric_config['workspace_id']
            lakehouse_id = fabric_config['lakehouse_id']
            shortcut_connection_id = fabric_config['shortcut_connection_id']
            #shortcut_name = fabric_config['shortcut_name']
            skip_if_shortcut_exists = True #fabric_config['skip_if_shortcut_exists']
        
            max_retries = 3
            max_threads = 2
        
            def create_shortcut(table):        
                nonlocal sc_created, sc_failed, sc_skipped
                catalog_name = table['catalog_name']
                schema_name = table['schema_name']
                operation = table['operation']
                #table_name = shortcut_name.format(schema=schema_name, table=table['name'], catalog=catalog_name)
                #table_name = f"{schema_name}_{table['name']}"
                table_name = f"{table['name']}"
                table_type = table['table_type']
        
                if operation == "create":
        
                    if table_type in {"EXTERNAL"}:
        
                        data_source_format = table['data_source_format']
                        table_location = table['storage_location']
        
                        if data_source_format == "DELTA":
                            # Remove the 'abfss://' scheme from the path
                            without_scheme = table_location.replace("abfss://", "", 1)
        
                            # Extract the storage account name and the rest of the path
                            container_end = without_scheme.find("@")
                            container = without_scheme[:container_end]
                            remainder = without_scheme[container_end + 1:]
        
                            account_end = remainder.find("/")
                            storage_account = remainder[:account_end]
                            path = remainder[account_end + 1:]
                            https_path = f"https://{storage_account}/{container}"
        
                            url = f"https://{FABRIC_API_ENDPOINT}/v1/workspaces/{workspace_id}/items/{lakehouse_id}/shortcuts"
        
                            if not skip_if_shortcut_exists:
                                url += "?shortcutConflictPolicy=GenerateUniqueName"
        
                            token = user_identity_bearer_token
        
                            payload = {
                                "path": f"Tables/{schema_name}",
                                "name": table_name,
                                "target": {
                                    "adlsGen2": {
                                        "location": https_path,
                                        "subpath": path,
                                        "connectionId": shortcut_connection_id
                                    }
                                }
                            }
                            headers = {
                                'Authorization': f'Bearer {token}',
                                'Content-Type': 'application/json',
                            }
        
                            for attempt in range(max_retries):
                                try:
                                    response = requests.post(url, json=payload, headers=headers)
                                    if response.status_code == 429:
                                        retry_after_seconds = 60
                                        if 'Retry-After' in response.headers:
                                            retry_after_seconds = int(response.headers['Retry-After']) + 5
                                        print(f"! Upps [429] Exceeded the amount of calls while creating '{table_name}', sleeping for {retry_after_seconds} seconds.")
                                        time.sleep(retry_after_seconds)
                                    elif response.status_code in [200, 201]:
                                        data = json.loads(response.text)
                                        print(f"Shortcut created successfully with name:'{data['name']}'")
                                        sc_created += 1
                                        break
                                    elif response.status_code == 400:
                                        data = json.loads(response.text)
                                        error_details = data.get('moreDetails', [])
                                        error_message = error_details[0].get('message', 'No error message found')
                                        if error_message == "Copy, Rename or Update of shortcuts are not supported by OneLake.":
                                            print(f"Skipped shortcut creation for '{table_name}'. Shortcut with the SAME NAME exists.")
                                            sc_skipped += 1
                                            break
                                        elif "Unauthorized. Access to target location" in error_message:
                                            print(f"status_code [400] Cannot create shortcut for '{table_name}'. Access denied, please review.")
                                        else:
                                            print(f"! Upps [{response.status_code}] Failed to create shortcut. Response details: {response.text}")
                                    elif response.status_code == 403:
                                        print(f"! Upps [403] Cannot create shortcut for '{table_name}'. Access forbidden, please review.")
                                    else:
                                        print(f"! Upps [{response.status_code}] Failed to create shortcut '{table_name}'. Response details: {response.text}")
                                except requests.RequestException as e:
                                    print(f"Request failed: {e}")
                                if attempt < 2:
                                    sleep_time = 2 ** attempt
                                    print(f"___ Retrying in {sleep_time} seconds for '{table_name}'...")
                                    time.sleep(sleep_time)
                                else:
                                    print(f"! Max retries reached for '{table_name}'. Exiting.")
                                    sc_failed += 1
                                    break
                        else:
                            print(f"Skipped shortcut creation for '{table_name}'. Format not supported: {data_source_format}.")
                            sc_skipped += 1
                    else:
                        print(f"∟ Skipped shortcut creation for '{table_name}'. Table type is not EXTERNAL.")
                        sc_skipped += 1
                else:
                    print(f"∟ Skipped shortcut creation for '{table_name}'. Shortcut with the SAME NAME exists.")
                    sc_skipped += 1 
                
            with ThreadPoolExecutor(max_workers=max_threads) as executor:
                executor.map(create_shortcut, tables)
            
            return sc_created, sc_skipped, sc_failed
        ```
        
3. Finally using the below script you can execute to automate the shortcut creation in fabric Lakehouse.
    

```python
# Databricks workspace
dbx_workspace = your_databricks_workspace
dbx_token = your_databricks_pat_token

# Unity Catalog
dbx_uc_catalog = "databricks_catalog_name"
dbx_uc_schemas = '["schema1", "schema2","schema3"]'

# Fabric
fab_workspace_id = "fabric_workspace_id"
fab_lakehouse_id = "fabric_lakehouse_id"
fab_shortcut_connection_id = "cloud_connection_id"
# If True, UC table renames and deletes will be considered
fab_consider_dbx_uc_table_changes = True

databricks_config = {
    'dbx_workspace': dbx_workspace,
    'dbx_token': dbx_token,
    'dbx_uc_catalog': dbx_uc_catalog,
    'dbx_uc_schemas': json.loads(dbx_uc_schemas)
}

fabric_config = {
    'workspace_id': fab_workspace_id,
    'lakehouse_id': fab_lakehouse_id,
    'shortcut_connection_id': fab_shortcut_connection_id,
    "consider_dbx_uc_table_changes": fab_consider_dbx_uc_table_changes
}

dbr_uc_tables_to_onelake(databricks_config, fabric_config)
```

### Consideration

* You need to have a schema enabled lakehouse in fabric which is currently at public preview
    
* You can use Key Vault Integration for managing the secrets of databricks PAT
    
* As fabric table shortcuts only supports Delta tables so the external tables those are non Delta tables will not be migrated into fabric lakehouse
    
* If you have multiple storage account involved for your unity catalog then you need to run the notebook separately
    
* As we are doing this operation from Databricks workspace you need to add fabric Service Principal and secrets to set spark configs to access OneLake DFS endpoints as mentioned in the attached code