# AWS Serverless Web App --- Student Data Management

A beginner-friendly serverless web application that lets users **save
student details and view the student list** from a browser.

**Repository:** `AWS-SERVEERLESS-WEB-APP`\
**AWS Region shown in the project screenshots:** `ap-south-1` (Asia
Pacific --- Mumbai)

> Replace placeholders such as `<YOUR_API_INVOKE_URL>` and
> `<YOUR_BUCKET_NAME>` with values from your own AWS account. Do not
> commit AWS access keys, secret keys, or other credentials to GitHub.

![image alt](https://github.com/Y-Vishanth/Aws-Serverless-Web-App/blob/c4703147a0fac8d41871298d6d214a42cbf82912/Website%20Pic.jpeg)

## 1. Project overview

The application has a simple web interface with fields for:

-   Student ID
-   Name
-   Class
-   Age

The user can select **Save Student Data** to store a student record and
**View all Students** to retrieve and display the records.

### AWS services used

  -----------------------------------------------------------------------
  Service                             What it does in this project
  ----------------------------------- -----------------------------------
  **Amazon S3**                       Hosts the static frontend files
                                      (`index.html` and `scripts.js`).

  **Amazon API Gateway**              Provides HTTP endpoints that the
                                      frontend calls. The screenshots
                                      show `GET`, `POST`, and `OPTIONS`
                                      methods on the root resource.

  **AWS Lambda**                      Runs Python code when the API is
                                      called. The visible function is
                                      `get-student-data`; the project
                                      also needs a write operation for
                                      saving records.

  **Amazon DynamoDB**                 Stores student records in the
                                      `student-data` table.
  -----------------------------------------------------------------------

## 2. Architecture

``` text
User's browser
     |
     | Loads the website
     v
Amazon S3 static website
(index.html + scripts.js)
     |
     | HTTPS API request (GET / POST)
     v
Amazon API Gateway
     |
     +----------------------+
     |                      |
     v                      v
GET Lambda              POST Lambda
(read student list)     (save a student)
     |                      |
     +-----------+----------+
                 |
                 v
        Amazon DynamoDB
          student-data
                 |
                 v
       Results returned as JSON
                 |
                 v
     Browser displays student list
```

**Important:** API Gateway does not automatically connect to DynamoDB.
It invokes the configured Lambda function, and that function uses its
IAM execution role to access DynamoDB.

## 3. Repository structure

A simple repository structure is:

``` text
AWS-SERVEERLESS-WEB-APP/
├── frontend/
│   ├── index.html
│   └── scripts.js
├── lambda/
│   ├── get_student_data/
│   │   └── lambda_function.py
│   └── save_student_data/
│       └── lambda_function.py
└── README.md
```

If your current files are stored at the repository root, you can keep
them there. The structure above is a suggested way to organize the
project, not a requirement imposed by AWS.

## 4. Prerequisites

You need:

1.  An AWS account.
2.  Permission to create S3 buckets, DynamoDB tables, Lambda functions,
    API Gateway APIs, and IAM roles/policies.
3.  A browser and a text editor.
4.  Basic familiarity with HTML, JavaScript, and Python is helpful, but
    not required to follow the deployment steps.

Use one AWS Region consistently. The screenshots show `ap-south-1`;
select that region in the AWS console unless you intentionally deployed
resources elsewhere.

## 5. Step 1 --- Create the DynamoDB table

1.  Open the AWS Management Console and go to **Amazon DynamoDB**.

2.  Select **Tables → Create table**.

3.  Enter the table name:

    ``` text
    student-data
    ```

4.  Set the partition key to:

    ``` text
    studentid
    ```

5.  Select the key type **String**.

6.  Keep the remaining settings appropriate for a learning project and
    create the table.

### Why is `studentid` the partition key?

DynamoDB needs a primary key to identify each item. In this project,
`studentid` identifies a student record. Each item can also contain
attributes such as `name`, `class`, and `age`.

Example item:

``` json
{
  "studentid": "A-001",
  "name": "Ramesh",
  "class": 10,
  "age": 17
}
```

The spelling and capitalization of attribute names must match the names
used in your Lambda code. The screenshots show `studentid`, `name`,
`class`, and `age`.

## 6. Step 2 --- Create the Lambda functions

The project uses Lambda to run backend code without managing a server.

### A. Function to retrieve students

1.  Open **AWS Lambda → Create function**.
2.  Choose **Author from scratch**.
3.  Enter a name such as `get-student-data`.
4.  Choose **Python** as the runtime.
5.  Create the function.
6.  In the code editor, add the logic to read items from the
    `student-data` table.
7.  Deploy the code.

The screenshot shows the function using `boto3`, selecting the
`student-data` table, and calling `table.scan()` to retrieve items. A
scan reads table items; for a large production table, review pagination,
capacity, cost, and whether a more targeted query is appropriate.

### B. Function to save a student

Create a second Python Lambda function, for example `save-student-data`.

Its logic should:

1.  Read the JSON request body sent by API Gateway.
2.  Parse the submitted student ID, name, class, and age.
3.  Validate that required values are present and that numeric fields
    contain valid numbers.
4.  Write the item to DynamoDB using `put_item`.
5.  Return a JSON response and an appropriate HTTP status code.

Illustrative core operation:

``` python
table.put_item(
    Item={
        "studentid": student_id,
        "name": name,
        "class": student_class,
        "age": age
    }
)
```

This is the core DynamoDB write operation, not a complete Lambda
handler. Your handler must parse the API Gateway event, validate inputs,
handle exceptions, and return the response format expected by your API.

### C. Give Lambda permission to access DynamoDB

Each Lambda function needs an **execution role** with the permissions it
requires.

For a learning project, create or update the function's execution role
to allow only the required actions on the `student-data` table:

-   Read function: `dynamodb:Scan` (and any other read actions actually
    used).
-   Write function: `dynamodb:PutItem`.

Restrict the policy's `Resource` to the ARN of your table. Avoid
granting broad `dynamodb:*` permissions on all tables. The Lambda
execution role is separate from the permissions that allow API Gateway
to invoke Lambda.

## 7. Step 3 --- Create API Gateway endpoints

The screenshots show an API named `student`, a root resource `/`, and
`GET`, `POST`, and `OPTIONS` methods.

1.  Open **Amazon API Gateway**.
2.  Create or open your API.
3.  Select the root resource `/`.
4.  Create a **GET** method and configure it to invoke
    `get-student-data`.
5.  Create a **POST** method and configure it to invoke your
    save-student Lambda function.
6.  Ensure API Gateway has permission to invoke each Lambda function.
    The console can add the required Lambda resource permission when you
    configure the integration.
7.  Configure CORS for the API. Allow the frontend origin and the
    methods and headers your JavaScript actually sends. For a public
    static S3 website, the origin is the website endpoint shown in the
    browser address bar.
8.  Deploy the API to a stage, for example `prod`.
9.  Copy the **Invoke URL** shown for that stage.

Your API URL will look similar to:

``` text
https://<api-id>.execute-api.<region>.amazonaws.com/<stage>
```

Use the actual Invoke URL from your API Gateway console; do not copy the
example literally.

### What do GET, POST, and OPTIONS mean?

-   **GET** retrieves the student list.
-   **POST** submits a new student record to be saved.
-   **OPTIONS** is commonly used by browsers for a CORS preflight
    request. API Gateway's CORS configuration must be consistent with
    the methods and headers the frontend uses.

After changing API methods or integrations, redeploy the API to the
stage before testing the public endpoint.

## 8. Step 4 --- Connect the frontend to API Gateway

The frontend consists of `index.html` and `scripts.js`.

1.  Open `scripts.js`.
2.  Find the API URL or endpoint constant used by the `fetch()` calls.
3.  Replace it with the API Gateway Invoke URL from your deployed stage.
4.  Ensure the save button sends a `POST` request with a JSON body.
5.  Ensure the view button sends a `GET` request and renders the
    returned items in the student table.
6.  Upload the updated frontend files to S3.

Conceptual JavaScript examples:

``` javascript
// Replace with your real API Gateway stage URL.
const API_URL = "https://<api-id>.execute-api.<region>.amazonaws.com/<stage>";
```

A save request typically sends JSON:

``` javascript
fetch(API_URL, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    studentid: "A-004",
    name: "Priya",
    class: 11,
    age: 16
  })
});
```

A read request typically calls:

``` javascript
fetch(API_URL, {
  method: "GET"
});
```

These snippets illustrate the request shape. Adapt them to your existing
`scripts.js`, including its response handling and any path suffixes you
configured. If your GET and POST methods use different resource paths,
set separate URLs accordingly.

## 9. Step 5 --- Host the frontend in Amazon S3

1.  Open **Amazon S3 → Create bucket**.
2.  Choose a globally unique bucket name, such as
    `my-student-app-<unique-suffix>`.
3.  Select the same Region used for your project, if possible.
4.  Upload `index.html` and `scripts.js` (or upload the contents of your
    `frontend/` directory).
5.  Open the bucket's **Properties** tab.
6.  Under **Static website hosting**, enable website hosting and set
    `index.html` as the index document.
7.  Copy the S3 website endpoint displayed by AWS and open it in your
    browser.

### Access and security note

An S3 static website endpoint serves content over HTTP. It is not HTTPS
by itself. For a more secure public website, consider using CloudFront
with HTTPS. If you enable public read access for a learning project,
restrict it to the frontend objects that must be public. Never put
credentials or private data in frontend JavaScript: anything delivered
to a browser can be inspected by users.

The browser needs permission to read the frontend objects. The Lambda
functions, not the browser, should access DynamoDB.

## 10. Step 6 --- Test the application

Test in this order:

1.  Open the S3 website endpoint.
2.  Enter a new student ID, name, class, and age.
3.  Click **Save Student Data**.
4.  Confirm the UI displays a success or error message.
5.  Open DynamoDB and inspect the `student-data` table to confirm the
    item was stored.
6.  Click **View all Students**.
7.  Confirm the saved item appears in the table.
8.  Check Lambda's **Monitor → View CloudWatch logs** if a request
    fails.
9.  Use the browser's Developer Tools → **Network** and **Console** tabs
    to inspect failed API requests and CORS errors.

### Expected result

The page should let a user save a student and retrieve the student list.
The screenshots show sample records such as `A-001`, `A-002`, and
`A-003`; your table will show whichever records you have actually
inserted.

## 11. Troubleshooting

### The page loads, but saving does not work

-   Confirm the POST URL and method in `scripts.js`.
-   Confirm API Gateway's POST integration points to the save Lambda
    function.
-   Check the Lambda execution role for `dynamodb:PutItem` on the
    correct table.
-   Inspect the POST response in browser Developer Tools and check
    Lambda CloudWatch logs.

### The student list does not load

-   Confirm the GET endpoint is correct and deployed.
-   Check the GET Lambda function's table name and Region.
-   Confirm its execution role permits the read operation it uses.
-   Inspect the response body and Lambda logs.

### CORS error in the browser

-   Configure CORS for the API resource.
-   Allow the actual S3 website origin, the required HTTP methods, and
    headers such as `Content-Type` when needed.
-   Ensure the OPTIONS preflight response is configured correctly.
-   Redeploy the API after changing configuration.
-   Check that Lambda responses include the required CORS headers if
    your integration setup requires them.

### DynamoDB table appears empty

-   Confirm the POST Lambda ran successfully.
-   Check the exact table name and Region.
-   Inspect the Lambda logs and the POST response.
-   Verify that the item uses the correct partition-key attribute name:
    `studentid`.

### Changes to frontend code are not visible

-   Confirm you uploaded the updated files to the correct bucket.
-   Refresh the browser. If you later use CloudFront, invalidate or wait
    for cached content as appropriate.

## 12. Cost and cleanup

This is a learning project, but AWS resources can still incur charges
depending on configuration and usage. Check the pricing and billing
pages for the services and settings you use.

When finished practicing:

1.  Disable or delete the API Gateway API if you no longer need it.
2.  Delete the Lambda functions if they are no longer needed.
3.  Delete the DynamoDB table after exporting any data you want to keep.
4.  Empty the S3 bucket, then delete it.
5.  Review IAM roles and policies created only for this project, and
    remove them if no longer needed.
6.  Check AWS Billing and Cost Management for any remaining resources or
    charges.

Do not delete shared roles, tables, or buckets used by other
applications.

## 13. What I learned

-   How a static frontend can be hosted in Amazon S3.
-   How API Gateway exposes HTTP endpoints.
-   How Lambda runs backend logic without managing servers.
-   How DynamoDB stores student records.
-   How frontend JavaScript sends requests to an API.
-   How IAM permissions and CORS affect a serverless application.
-   How to test and troubleshoot the complete request flow.

## 14. Future improvements

Possible next steps include:

-   Add input validation and clearer error messages.
-   Add update and delete operations.
-   Add pagination instead of scanning the entire table for larger
    datasets.
-   Add authentication and authorization before storing real student
    information.
-   Use CloudFront and HTTPS for frontend delivery.
-   Manage the resources with AWS SAM, AWS CDK, or Terraform.
-   Add automated tests and a CI/CD pipeline.

------------------------------------------------------------------------


