IMPORTANT ❗ ❗ ❗ Please remember to destroy all the resources after each work session. You can recreate infrastructure by creating new PR and merging it to master.
  
![img.png](doc/figures/destroy.png)

1. Authors:

   ***TBD Group 3***

   [***link to forked repo***](https://github.com/rafauoo/tbd-workshop-1)
   
2. Follow all steps in README.md.

3. In boostrap/variables.tf add your emails to variable "budget_channels".

    ![obraz](https://github.com/user-attachments/assets/99441767-61ae-4ab7-9cb9-f579d1cf9d8a)

4. From avaialble Github Actions select and run destroy on main branch.

    ![obraz](https://github.com/user-attachments/assets/7d5b5073-a85f-436f-aeb5-f76cac6c1c1a)

5. Create new git branch and:
    1. Modify tasks-phase1.md file.

       ![obraz](https://github.com/user-attachments/assets/505f6a80-cb8f-41a7-a227-7f4b3f114a3f)

    3. Create PR from this branch to **YOUR** master and merge it to make new release. 
    
    ![image](https://github.com/user-attachments/assets/468e5080-9f8e-47f5-b9ae-46c93112dc73)


6. Analyze terraform code. Play with terraform plan, terraform graph to investigate different modules.
Module: dataproc

This Terraform module provisions a Google Cloud Dataproc Cluster, which is a managed Apache Spark and Hadoop service. The module automates the setup of the Dataproc cluster with configurable parameters such as machine type, image version, region, and subnet. This allows users to quickly deploy and manage scalable Big Data workloads without needing to configure and maintain the infrastructure manually.By leveraging Google Cloud Dataproc, this module provides a cost-effective and easy-to-use solution for running distributed data processing jobs. The cluster can be used for machine learning, data analytics, ETL pipelines, and other compute-intensive tasks.
Resources Created

This module provisions the following resources in Google Cloud:

* Google Dataproc Cluster (google_dataproc_cluster.tbd-dataproc-cluster) - A fully managed cluster that supports Spark, Hadoop, Hive, and other Big Data tools.

* Google Project Service (google_project_service.dataproc) - Ensures that the Dataproc API is enabled in the project, allowing the creation and management of Dataproc clusters.

Graph:

    ```
    cd modules/dataproc
    terraform init -upgrade
    terraform graph -type=plan | dot -Tpng > terraform-plan-graph-dataproc.png
    ```
file:///home/dominika/tbd-workshop-1/modules/dataproc/terraform-plan-graph-dataproc.png

   
7. Reach YARN UI
   
   ```bash
    # Create an SSH tunnel using local port 8080
    gcloud compute ssh tbd-cluster-m \
    --project=tbd-2025l-318409 \
    --zone=europe-west1-d -- -D 8080 -N

    # Run Chrome and connect through the proxy (macOS)
    /usr/bin/google-chrome --proxy-server="socks5://localhost:8080" \
    --user-data-dir="/tmp/tbd-cluster-m" http://tbd-cluster-m:8088
    ```
   ![hadoop](https://github.com/user-attachments/assets/69d5bc16-3b51-4efd-8a1d-cd489212e543)

   
8. Draw an architecture diagram (e.g. in draw.io) that includes:
    1. VPC topology with service assignment to subnets
    2. Description of the components of service accounts
    3. List of buckets for disposal
    4. Description of network communication (ports, why it is necessary to specify the host for the driver) of Apache Spark running from Vertex AI Workbech

  ![image](https://github.com/user-attachments/assets/2a21dd1f-308d-46b3-8892-89b722b54f54)



9. Create a new PR and add costs by entering the expected consumption into Infracost
For all the resources of type: `google_artifact_registry`, `google_storage_bucket`, `google_service_networking_connection`
create a sample usage profiles and add it to the Infracost task in CI/CD pipeline. Usage file [example](https://github.com/infracost/infracost/blob/master/infracost-usage-example.yml) 
in progress
```
version: 0.1
resource_usage:
  google_artifact_registry.tbd_registry:
    storage_gb: 50

  module.data-pipelines.google_storage_bucket.tbd-code-bucket:
    storage_gb: 192                         
    monthly_class_a_operations: 1000000       
    monthly_class_b_operations: 12500000       
    monthly_egress_data_transfer_gb:
      same_continent: 550                   
      worldwide: 12500                        
      asia: 1500                              
      china: 50                              
      australia: 250

  module.gcp_mlflow_appengine.google_service_networking_connection.private_vpc_connection:
    monthly_egress_data_transfer_gb:
      same_region: 250                   
      us_or_canada: 100                     
      europe: 70                         
      asia: 50                           
      south_america: 100                   
      oceania: 50               
      worldwide: 200
```

   ![obraz](https://github.com/user-attachments/assets/ecd52e84-0788-4d3a-b5e0-9a2079b2c013)

10. Create a BigQuery dataset and an external table using SQL

    ```
    CREATE SCHEMA IF NOT EXISTS demo OPTIONS(location = 'europe-west1');

    CREATE OR REPLACE EXTERNAL TABLE demo.shakespeare
      OPTIONS (
    
      format = 'ORC',
      uris = ['gs://tbd-2025l-318409-data/data/*.orc']);
    
    
    SELECT * FROM demo.shakespeare ORDER BY sum_word_count DESC LIMIT 5;
    ```
    ![obraz](https://github.com/user-attachments/assets/636657a2-9a6b-4f31-8049-0135edd96b8b)

   
ORC doesn't require a table schema because it's a self-describing format that stores metadata about the schema within the file itself. This metadata includes column names, data types, and structure, allowing BigQuery to automatically infer the schema when creating an external table. Unlike formats like CSV that require explicit schema definitions, ORC's built-in schema information makes it more convenient for working with external data in BigQuery.

11. Find and correct the error in spark-job.py

    Error was easily visible in airflow logs so I changed path to correct project.
    
![image](https://github.com/user-attachments/assets/a9bad4c2-7b56-42a7-a5fb-cc0eb47c0e02)

  Result:
  
![obraz](https://github.com/user-attachments/assets/9f8f9398-df04-4173-bf75-5d3bd763c73f)

12. Add support for preemptible/spot instances in a Dataproc cluster

    Link: https://github.com/rafauoo/tbd-workshop-1/edit/master/modules/dataproc/main.tf
    ```
    resource "google_dataproc_cluster" "tbd-dataproc-cluster" {
      #checkov:skip=CKV_GCP_91: "Ensure Dataproc cluster is encrypted with Customer Supplied Encryption Keys (CSEK)"
      depends_on = [google_project_service.dataproc]
      name       = "tbd-cluster"
      project    = var.project_name
      region     = var.region
      ...
        preemptible_worker_config {
            num_instances = 1
        }
      }
    }
    ```
    ![image](https://github.com/user-attachments/assets/d4148932-745d-4454-a6f4-e8049c5ccfc1)
    
    The result:
    will be here

    
    
