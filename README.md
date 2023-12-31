## Hybrid Cloud Application Project

### Team Members:
1. **Satya Pranay Manas Nunna**
2. **Vraj Rana**
3. **Nishant Chaturvedi**

## Project 1: Elastic Web Application on AWS

### 1. Problem Statement
The objective of this project is to build an elastic web application that can automatically scale in and scale out based on demand in a cost-effective manner by using Amazon Web Services (AWS). The application provides image recognition as a service based on the number of requests received, and the number of server instances is scaled up or scaled down by using services provided by AWS.

### 2. Design and Implementation
#### 2.1 Architecture



**Architecture Description:**

As observed in the figure above, our architecture consists of 1 EC2 instance of Web Tier that takes multiple requests from a user in the system. These requests will be sent into the SQS request queue, which will be listened to by the App Tier EC2 instances (1 min, 19 max), and these image requests will be stored in the S3 bucket for persistence.

The SQS queue is monitored using CloudWatch to generate alerts, which will trigger an EC2 auto scaler to increase the number of instances based on the demand. The same CloudWatch alert is used to generate an alert to scale down the number of instances till the count of available instances equals the minimum requirement.

App Tier instances run a deep learning algorithm to predict output labels for the user-requested images by taking the requests from the SQS request queue. The predicted output label from the App Tier is stored in the S3 bucket as a (key, value) object for persistence, and the output is sent to the SQS response queue.

We have designed a dictionary in the Web Tier so that the output from the SQS response queue can be quickly pushed to the dictionary, and when the user requests for the results, the output gets deleted from the dictionary, which improves the overall efficiency of the architecture and simplifies the design.

**AWS Services:**
- AWS EC2: EC2 is being used to create the virtual machines that run the web tier and app tier, which are the instances being used in running the application. The instance images were built upon the LINUX image provided by the professor, which contained the deep learning model and environment pre-installed for implementing the NodeJS code related to the application.
- AWS SQS: SQS is being used in the project to maintain the request queue, namely SqsInputCloud, which is read by the app tier to process the images, and also a response queue, namely SqsOutputCloud, is being used to store the generated output responses generated by the app tier. SQS is used to make Web Tier and App Tier loosely coupled.
- AWS S3: S3 is being used in the project to store the data before and after the transformation of the request, i.e., input image is in the S3InputCloud bucket, and the text file contains the image file name and the classification output.
- CloudWatch: CloudWatch is being used in the project to monitor the SQS request queue to generate an alert for the AWS autoscaling group. Once the alert is generated, the auto-scaling group will assign a given number of instances to the group to handle the additional inflow of requests.

#### 2.2 Autoscaling

In the project, we are using CloudWatch to implement the auto-scaling required by the application. The procedure is as follows:
- The input queue is monitored to check the number of available messages. If the number continues to be above 5 for more than a minute, then CloudWatch will generate a trigger/alert for the AWS EC2 auto-scaling group, which in response will create the appropriate number of instances to handle the workload.
- Another alert is also created to check the requests in the queue, which will be triggered every one minute. If the number of requests is less than 5 in the queue, then an alert will be generated and sent to the auto-scaling group, which will terminate the instances till the number of instances reduces to 1.

### 3. Testing and Evaluation

To test the scalability of the application, we used the workload generator to generate 100 requests to our server. 

#### 3.1 Query
```bash
python3 multithread_workload_generator.py --num_request 100 --url http://100.25.10.233:3000/ --image_folder imagenet-100/
```

#### 3.2 State of the Instances
As the number of messages increased in the SQS queue, SqsInputCloud, the alarm to scale up the instances gets triggered, and in the figure below, we can see how the number of instances increased after the request was hit. Once the messages in the queue reduce, the instances' count starts reducing gradually.



#### 3.3 S3 Bucket s3inputcloud (To store all the input images)

All the 100 images are stored in the s3 bucket for the classifier to process them. Below image shows the state of the bucket after the requests.


#### 3.4 S3 Bucket s3outputcloud (To store all the classification results)

All the classification results for the 100 input images are stored in the S3 output bucket. Below image shows the state of the bucket after the requests.



### 4. Code and Installation

**App Tier**

The App Tier is the main server responsible for classifying the results of the input images. It is connected to the AWS auto-scaling group to increase/decrease the number of instances based on the demand.

**Code:**
- **Classifier**: This folder contains the Image classifier and imagenet-labels.json files. Image Classifier will be executed from app_tier.sh file after receiving the input from the SQS input queue and storing the input in the S3 input bucket.
- **Controller**: This folder contains receiver_app_tier.js and sender_app_tier.js files.
  - **receiver_app_tier.js**: This file fetches data from SQS input queue and stores it in the S3 input bucket.
  - **sender_app_tier.js**: This file runs after the image classifier. It stores output to the S3 output bucket, from where SQS output queue gets the output for the REST API.
- **app_tier_installation.sh**: This file contains commands to download all the modules required to run the App Tier.
- **app_tier.sh**: This file executes receiver_app_tier.js, image classifier, and sender_app_tier.js in order in every instance with the help of crontab to generate the output.

**Installation:**
1. Run the script app_tier_installation.sh to update all the dependencies.
2. Go to directory /app_tier/ and run the command `npm install` to update all the node modules.
3. Create a cron tab using `crontab -e` and write the below cron schedule:

```cron
* * * * * sh /home/ubuntu/app_tier/app_tier.sh
* * * * * sleep 10; sh /home/ubuntu/app_t