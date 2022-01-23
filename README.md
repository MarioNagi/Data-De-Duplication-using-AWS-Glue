# Data-De-Duplication-using-AWS-Glue


###Problem description:

The following example of the dataset1 and dataset 2 shows how it’s structured. The dataset1 includes 2,616 records, structured as shown in the following table.
![image](https://user-images.githubusercontent.com/22025520/150676041-7ffd1121-48aa-4435-a1c9-a303f591b0ef.png)
The dataset 2 includes 64,263 records. It has a similar structure, but the data is messier. For instance, it has missing entries, incorrect values (for example, an address in the title field), and includes unexpected characters.
![image](https://user-images.githubusercontent.com/22025520/150676046-0bbd023b-48e1-452e-987c-e3296add2fd2.png)
Using the provided labels file, shown below, we need to find the deduplication data using AWS services

###Solution steps:
1)	The 1st step to do it to merge the 2 datasets together, and to avoid compatibility issues, replace all “/” characters in the “id” field with “-” in both the datasets and label file and adding new column called source to represent from which the dataset. The next step to do is to upload the combined dataset to S3 to start working on it.
![image](https://user-images.githubusercontent.com/22025520/150676071-10779e78-6a27-408a-8246-352d8b6a150a.png)
![image](https://user-images.githubusercontent.com/22025520/150676073-cabe0542-2176-49e9-b740-6317d2d4fddf.png)
2)	In this exercise Glue Data Catalog (crawler) will be used to create catalog the data so we can other services on it. You have to create or use a new Role to enable Glue access to S3
the best thing about Glue is that you don’t need to move the data from S3, the Glue will work on the data on S3
![image](https://user-images.githubusercontent.com/22025520/150676076-a4aa7c3d-b6dc-4f5b-8bba-4667aa1fdbe9.png)
After the crawler finishes running, it will create a table with the following schema 	
![image](https://user-images.githubusercontent.com/22025520/150676085-6f7c9284-83ce-4f52-bdad-044a661f3e9e.png)
3)	Next we will use create a new FindMatches ML transform for your data. With the following configurations
![image](https://user-images.githubusercontent.com/22025520/150676088-73d4a3d8-f79d-4aaf-8ab3-dfab7a3635e1.png)
Then choosing the data from S3, after that we have to pick a primary key, select id as the primary key. Since the primary key should be a unique identifier
![image](https://user-images.githubusercontent.com/22025520/150676100-df413c22-1c28-4001-8405-a534afb8b1d9.png)
![image](https://user-images.githubusercontent.com/22025520/150676102-3da83eda-bc0a-4b18-9972-e752dcbf90dd.png)
On the tune transform page, you have to adjust the balance between Recall and Precision, and between Lower cost and Accuracy.
Based on your choice, the cost and the accuracy will be determined, in this example We will favor accuracy over cost, so we will move the 1st  slider towards Precision (0.9) and 2nd towards Accuracy (1.0)
![image](https://user-images.githubusercontent.com/22025520/150676105-663a9e26-2956-4aac-9885-ac9e716f3ac8.png)
Now we need to teach the model with train data, if we tried to upload the normal label file we have on hand. It will be rejected as it need to adhere to the following format:
“ labeling_set_id, label, id, title, ... “
you can create it using the generated option in transformML or creating it manually
we will divide the labels into sets which will be added in the labeling set id column, and the label column will be used to mark the duplicated books as seen in the below screenshot
![image](https://user-images.githubusercontent.com/22025520/150676108-cc2bfac3-d2ee-428f-bbaf-2ac7ca8f0b54.png)
At least 100 records are needed to get accurate data, the more records will be added the better results will be achieved  
After uploading and teaching the model, accuracy estimation will be calculated 
![image](https://user-images.githubusercontent.com/22025520/150676116-59eb8f95-f154-4409-b7dc-6a473bf4bf53.png)
4)	After that we to create and run a record-matching job, in this job Glue will generate a spark script which will use the transform job to identify the duplicated data
![image](https://user-images.githubusercontent.com/22025520/150676133-a9ed6a02-5516-4b4d-b5ef-e594c8e1cccf.png)
In Security configuration, script libraries, and job parameters (optional), we have to pick “G.2X (Recommended for jobs with ML transforms)” to be able to process the transform ML created earlier
![image](https://user-images.githubusercontent.com/22025520/150676141-247f93bc-1b74-485a-b8cb-5dde566c8275.png)
On the next page, select the table within which to find matching records.
![image](https://user-images.githubusercontent.com/22025520/150676145-a15d3d7b-1b4c-43d0-b6ba-c6299c6977c7.png)
On the next page, select Find matching records as the transform type. To review records identified as duplicate, do not select Remove duplicate records. Choose Next.
![image](https://user-images.githubusercontent.com/22025520/150676150-8de35efd-e68f-4d2f-9ac2-16eb1181625e.png)
Next, select the transform that you created and choose Next.
![image](https://user-images.githubusercontent.com/22025520/150676156-eaf4fe17-e480-4198-b13c-1ce5d35e6011.png)
On the next page, select Create tables in your data target. For Data store, select Amazon S3. For Format, choose CSV. For Target path, choose a path for the job’s output. Choose Save job and edit script.
![image](https://user-images.githubusercontent.com/22025520/150676162-96339e4a-8101-4955-a311-f3dcfe38e19c.png)
After finishing the configuration, glue will create automatic spark script to do the required find matches
![image](https://user-images.githubusercontent.com/22025520/150676169-42caf120-cfee-4949-acd0-f4b23f2f7708.png)
The script will take around 10 mins to fully run and will create output files in the specified output location in S3 picked in the configuration.
The output will look something like this (“multipart CSV files”)
![image](https://user-images.githubusercontent.com/22025520/150676179-660b4f2b-d5bf-4ea5-8f14-08677add1dcd.png)
5)	In the final step, we will download these files via AWS CLI, to be able to use AWS CLI you must install on your system first. -install it from here https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html -
to check if the cli is installed correctly on your system, write “aws” in your command prompt, you should receive a similar screen 
![image](https://user-images.githubusercontent.com/22025520/150676190-54d7ff7e-a7bf-4de4-8da6-42cbb8e788fa.png)
To interact with your AWS account, you have to use the following command “aws configure” and enter your access key id and secret -if you don’t have access key, you can create one from AWS IAM-
To configure your session, you have to enter your access key ID, access key secret and your region -here I am using us-west-2- and the output format like the below image
![image](https://user-images.githubusercontent.com/22025520/150676193-00dfcc7b-6ecb-44e4-baf2-5f2338748515.png)
After that, you must ensure that your account has access to S3 through IAM policy otherwise you will receive an error while trying to interact with S3.
You can either use “aws sync” command or “aws cp – recursive” to download all the file in your output folder, then merge them to get one final CSV file as seen below a new column has been added named match_id which contains all the matched books

![image](https://user-images.githubusercontent.com/22025520/150676210-aeca928b-6d19-437a-8c1b-d0ce0415d72c.png)


###Used services:

       AWS Glue is a serverless data integration service that makes it easy to discover, prepare, and combine data for analytics, machine learning, and application development. AWS Glue provides all the capabilities needed for data integration so that you can start analyzing your data and putting it to use in minutes instead of months.
The beauty about Glue lies in its serverless operation, as we saw in the above example, Glue did the catalogue, transform Ml, transform job almost without any human interaction which could help users who has little or no experience with machine learning or spark 












