# document-manager

## System design goals
---

The overall goal of the system's design is to achieve a level of modularity where any single component of the system can potentially be replaced, if needed. 

One such system component could potentially be the Image Processing lambdas, in case the computing vision business logic required to run on a single document exceeds the maximum Lambda platform quota of 15 minutes per invocation. If the computing vision tasks are limited to text/image recognition then I'd highly suggest using a cloud native, managed AWS ML service like Textract or Rekognition.

Another component could be the email sending part of the system which could be replaced with any third-party provider, like Twilio or MailChimp, or indeed any further designs of our own. 

At the same time, this system is stateless and fault tolerant; error handling is built into it by storing lambda invocations that encountered exceptions during execution to a SQS Dead-Letter-Queue (DLQ) and such faults will not prevent the system from reaching it's end state because they'll be isolated to a particular document.

Last, but not least, an AWS billing analysis would indicate that this system costs no more of a handful dollars per month.

## Assumptions
---

The `FileLocation` property of documents stored in DocumentDB will need to include customer related data as part of the S3 object location so that we may know who it is we'll be sending the email to. This is a common pattern of data persistence & retrieval in blob storage.

## Image Processing Layer
---

The system kicks off based on an EventBridge Rule triggering a Lambda function once per week.

This first Lambda function acts as an aggregator, pulling last week's data needed for the report summaries, like the required documents from DocumentDB and files from S3 buckets as necessary.

For each such unique combination of data, a single report will be generated.

The Lambda function will publish a single message to a SNS topic for each report.

Another Lambda function, one with multiple concurrency enabled, is subscribed to this SNS topic and will invoke a Lambda for each such message. This is by default limited to 1000 due to platform quota but it's possible to request more from AWS up to hundreds of thousands.

Each Lambda will store the results of it's image processing to an S3 bucket.

## Reporting Layer
---

Due to the parallel and asynchronous nature of the image processing lambdas storing their respective results to S3, we cannot use a S3 PUT event as a lambda trigger for this next step for we cannot guarantee that all lambdas have ran to completion just yet.

Instead, an hour later, another EventBridge Rule will trigger a Lambda function that will aggregate the image processing results from the aforementioned S3 bucket, generate the .pdf report and do the email templating work (I imagine some personalization will take place here) before invoking SES to deliver those to their respective recipients.

I chose SES because it looks like a relatively safe choice (and it should certainly be the fastest one from a development implementation PoV) for this particular use case, because we already have a list of recipients who are subscribed to receiving emails from our email domain. That being said, I should also point out that guaranteed delivery (instead of ending to spam) might be jeopardized, using SES, if the amount of recipients grows beyond certain limits but there are workarounds such as throttling SES requests using an SQS queue.
