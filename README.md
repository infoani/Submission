API Documentation

# /initiate API

Method of invocation 
# curl -X POST http://127.0.0.1:6543/initiate -d '{"start_date": "2020-02-12", "end_date" : "2020-02-14"}'

Passing the start_date and end_date in body would initiate merge request of the log parts 2020-02-12_0, 2020-02-12_1 .. 2020-02-14_0, 2020-02-14_1 etc. 

Implementation and Processing details:

1. The request body would go through validation to check correct date format or if start date is less than end date. 
2. Passing the validation check would then invoke collection of file parts from AWS S3 (boto3)
3. The file parts after collection would be sorted cronologically in 2020-02-12_0 -> 2020-02-12_1 -> 2020-02-12_2 in this sequence. And the streaming object readers to each file handle would be stored in a python list.
4. Finally the program would go through each file and store each datestamp into a hash and values would be the content of the line. The sorting of the lines would be based on cronology. This sorting would have to be implemented in O(nlogn). This step is required since the content inside the file parts themselved don't follow cronological order. Then the file would be written back to local mount '/var/log/merged_logs'. The whole time complexity of the above process would be O(nlogn).
5. The above process would be repeated for each date in the date range passed. This would minimize the memory usage.
6. merge_s3_files function would take this merging request asyncronously with the help of celery and finally put the result in the redis backend.
7. The task request when created using the api would register the task in the redis database. Since it can take a while to merge large files, celery won't put the results back to redis till it's finished. Till then we would have no way to seperate invalid download ids with the ones that are still processing. Registering would ensure that requests with invalid download ids are prevented at the first check.
8. After the files are merged and stored in '/var/log/merged_logs'. They would then be uploaded back to S3, removing the local instance.
9. The redis backend doesn't store the whole content of the merged file but only the name using which it can be identified later. This is done to keep the RAM usage to a minimum. Since redis is a fully in-memory key value store database, the space in db can fill up pretty fast.
10. We would then read the file name using the calery task id, and the file name would be searched in the merged bucket of S3 to get back the content and process further.
11. When the task is created using /initiate API. It returns 201 (Created) return code along with the download_id (uuid) key.


# /download API

Method of invocation:
# curl -X GET http://127.0.0.1:6543/download?id=<download_id>

Implementation and processing details:

1. The download id received would be the key to retrieving the downloaded data.
2. The download API is checked for validation of parameters like if the download id is passed or if it's valid. 
3. We can query the redis db and check if the download id has been registered or not.
4. Invalid download ids would return a status code of 404.
5. If the download id is registered then the API would return a status code 202 (accepted) till the merging is finished.
6. Based on the download id the process would then get the file name associated with it and query the S3 bucket for the content.
7. Once the process gets back the content it would then return it back as a FileRespose object with status code 200 (success).

# test cases

7 test cases have been written using pytest to validate the api response. 6 of them are validation of wrong inputs. 7th one tests the whole journey of placing the request to downloading the file.
