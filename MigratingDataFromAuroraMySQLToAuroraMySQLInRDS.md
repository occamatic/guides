# Migrating data between two Aurora MySQL instances in RDS using AWS Database Migration Service (DMS)

As a prerequisite for migrating from a MySQL Aurora database, we need to apply a different binary logging format to the source RDS cluster that the database instance is a part of. To do this we need to first create a parameter group using the default.aurora5.6 cluster parameter group as a template.

![image](https://user-images.githubusercontent.com/70777733/143592572-b3d1ef25-99f6-47be-848e-32c8db369d6a.png)

Here I have called it “dms-test-parameter-group”. Within the parameter group you can search for the parameter “binlog_format” which we will change from “OFF” to “ROW”.

![image](https://user-images.githubusercontent.com/70777733/143592672-975a7219-f3c9-4d67-955c-d4f0b4c7d02a.png)

Apply this to the source database cluster and make sure to reboot the instance for the full change to take effect.

![image](https://user-images.githubusercontent.com/70777733/143592736-56512262-6cf7-4a31-8770-3fe1ae00487f.png)

Now to create the parts needed for the database migration. Go to AWS’ Database Migration Service (DMS) and create the DMS replication instance. 
Because we are using this for initial test purposes, it will be the smallest instance size possible which is dms.t3.micro. This instance will also have most configurations set to the default options, such as having the latest DMS engine version, 50 GB of storage etc. 
Having this size of replication instance with 50 GB of storage means that it still comes within the AWS free tier which makes it great for a test workload like this. 
Other settings included the VPC that the replication instance would run in, using single AZ as the source database is only in a single AZ and this is perfect for a dev or test workload as stated by the DMS console. The replication instance will be in the eu-west-1c availability zone as this is where our source database is also located. This is to reduce actual data transmission, to make sure that the replication has as little latency as possible.

For our source and target endpoints we can start easily as both are already hosted in RDS, it is simple to select the correct instance from the dropdown provided. This also allows DMS to automatically know the underlying database engine used for each instance, so that field will be pre-populated. 
As we do not store our access credentials in AWS Secrets Manager, we must select the option to “Provide access information manually”, where we enter in the username and password that we would usually use to access each database. Leaving everything else as default we can then test the connection from within the VPC, when successful this allows the endpoint creation to be complete. This is the same for both source and target database instances.

Now that our source and target endpoints are in place, we can create the migration task between the two. In our case we need to select the replication instance and both endpoints from their respective dropdowns, we also want a full load and to replicate ongoing changes from the source database to the target.
 
To preserve columns that auto-increment, we have to export the schema from the source database into the target database. DMS tries its best to migrate the data and the schema structure, including primary keys and constraints, but sometimes this means that certain details are missed out. Exporting the current schema(s) to CREATE statements can help overcome any issues that arise from this. 
Without copying the schema(s) across first, auto-increment columns may restart meaning that errors can arise from conflicts especially on values in these fields that may have a unique constraint. Exporting and importing schemas is very easy for this case as we have two tables but can become more complex with schemas with more tables. MySQL Workbench provides an easy right click option to make CREATE statements out of schemas and tables.
 
![image](https://user-images.githubusercontent.com/70777733/143592790-ab03ac1c-9997-4cb0-9e65-0848d06c3e15.png)

Now that we have already created the tables on the target database, we want to only truncate them as our migration task preparation mode. Dropping the tables at this point will make the last step pointless and subject us to the issues previously mentioned. We don’t want to stop the task as we don’t have cached changes to apply, we also want to use full LOB mode with the default chunk size. Although we do not have any columns with LOBs in this migration, this will be a good default going forward. Enable validation and CloudWatch Logs and leave the rest of the options as default until the “Table mapping” section. 

For table mappings, we only need a simple selection rule as we are migrating all data that doesn’t need transforming in any way. To avoid attempting to migrate system level tables that the database engine deals with, enter the schema name(s) of the databases for the migration. Use the wildcard of % for all tables within the schema.

We would like to run a premigration assessment as we would like to avoid any migration issues when using DMS. Before this can be enabled, we need to make sure that we have an S3 bucket to store the premigration report and an appropriate IAM role that can write the report to the bucket. 

For this example, I am creating a new bucket with all default settings to store the premigration report.
I also created an IAM role called “dms_role” that has full access to write to the S3 bucket.


Put these relevant values into the configuration of the premigration assessment and leave everything else as defaults and the migration task is ready to be created.

The premigration assessment report can be viewed like below, it should have passed all the checks so we are ready to run the migration task.

![image](https://user-images.githubusercontent.com/70777733/143592936-413e56d2-e62c-48c1-82af-6afc0f8678e7.png)

Start the migration task, it will update you with the status and even list the statistics for UPDATEs, INSERTs etc. on the source database and when they are complete on the target.

Once happy with the migration, the connection settings used by the applications can be changed to that of the target database.

After changing the settings for the applications, test them to make sure everything is working as usual and once confident that everything is working as intended you can proceed to shut down the source database. Shutting down is advised here in the off chance that all the steps above need to be reverted, completely deleting the source database at this point would mean that these actions cannot be reversed. A RDS instance that is shut down doesn’t incur any further charges. It is suggested that the applications run for a couple of days using the new database and connection settings before making the irreversible step.

After a couple of days, when happy that the migration did not cause any side effects, the source database can be terminated and the instance deleted. If there is a need to keep the instance shut down longer than 7 days, please be aware that it will automatically restart after 7 days of being shut down.
