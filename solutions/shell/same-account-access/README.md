# Shell script

This shell script by using AWS CLI goes through neccessary authorization code authentication flow required by data accessor (ISV) to access cross-account Q index data via Search Relevant Content API. 

## Prerequisites

- AWS CLI (v2) installed and configured on your computer

- Single AWS Accounts (One account running Amazon Q Business)
- Amazon Q Business application setup with IAM IDC as access management on AWS account 
- This sample uses Okta IdP instance with IAM Identity Center instance, but the sample principles and steps apply to any other OIDC-compliant external identity provider synced with IAM Identity Center.
- Enable Nova Pro model access on Amazon Bedrock

## Key Components

The key component of this solution is to show the user authentication flow step-by-step (OIDC authentication, token generation and management, STS credential handling) required to make Amazon Q Business's [SearchRelevantContent API](https://docs.aws.amazon.com/amazonq/latest/api-reference/API_SearchRelevantContent.html) requests to the same account's Q index. This is temporary solution for ISV's to be able to test accessing Q index of their own environment while waiting for the [data accessor registration process](https://docs.aws.amazon.com/amazonq/latest/qbusiness-ug/isv-info-to-provide.html) to complete.


## Usage Steps

### Setup Okta

1. Sign into Okta and go to the admin console.
2. In the left navigation pane, choose **Applications**, and then choose **Create App Integration**.
3. On the **Create a new app integration**** page, do the following:
  - Choose **OIDC – OpenID Connect**.
  - Choose **Web application**.
  - Then, choose **Next**.
4. On the **New Web App Integration** page, do the following:
  - In **General Settings**, for **App name**, enter a name for the application.
  - In **Grant type**, for **Core grants**, ensure that **Authorization Code** is selected. Expand **Advanced** and select on **Implicit (hybrid)** and **Allow Access Token with implicit grant type**.
  - In **Sign-in-redirect URIs**, add a URL that Okta will send the authentication response and ID token for the user's sign in request.
  - In **Assignements** > **Controlled access**, select **Allow everyone ins your organization to access**
  - Then, select **Save**.
5. From the application summary page, from General, do the following:
  - From **Client Credentials**, copy and save the **Client ID**. You will input this as the **Audience** when you create an identity provider in AWS Identity and Access Management in the next step.
6. From the left navigation menu, select **Security**, and then select **API**.
7. Then, from **Authorization Servers**, do the following:
  - Copy the **Issuer URI**, for example https://trial-okta-instance-id.okta.com/oauth2/default, and **Audience**. You will need to input this value as the **Provider URL** when you add your identity provider in IAM in Step 2.

### Setup Trusted Token Issuer

1. Sign into AWS Management Console, and go to **IAM Identity Center**
2. In the left navigation pane, choose Application assignements > Applications
3. Choose **Add application**
  - Select **I have an application I want to set up**
  - Application type as **OAuth 2.0**
  - Choose **Next**
  - In **Display name**, enter the name of this application
  - Select **Do not require assignements**
  - In **Application visiblity in AWS access portal**, select **Not visible**
  - Choose **Next**
  - In **Authentication with trusted token issuer**, select **Create trusted token issuer**
  - In **Issuer URL**, enter the Issuer URI retrieved from Okta
  - In **Trusted token issuer name**, enter the name of your application
  - Choose **Create trusted token issuer**
  - Go back to the previous window and click **refresh** to see the issuer just created
  - Select the **issuer** and in **Aud claim** enter the value retrieved from Okta
  - Choose **Next**
  - Select **Edit the application policy**
  - Enter the following policy:
    ```
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "Service": "qbusiness.amazonaws.com",
            "AWS": "<your-aws-account-id>"
          },
          "Action": [
            "sso-oauth:CreateTokenWithIAM"
          ],
          "Resource": [
            "<your-identity-center-custom-application-arn>"
          ]
        }
      ]
    }
    ```
  - Choose **Next**
  - Choose **Submit**

### Provide required information for data accessor in the shell script
- **ISSUER_URL** : Issuer URL of the IDP
- **IDP_CLIENT_ID** : Client ID of the IDP
- **REDIRECT_URL** : Callback URL that will provide authentication code
- **IAM_ROLE** : IAM Role ARN of the data accessor
- **QBUSINESS_APPLICATION_ID** : QBiz application ID of the enterprise account
- **RETRIEVER_ID** : Retrieval ID of the above QBiz application
- **IDC_APPLICATION_ARN** : ARN provided on data accessor configuration

![Configuration](assets/shell-configuration.png)

### Run the shell script
```
# ./data-accessor-tester.sh                                                                                           [/
Enter your prompt (or 'exit' to quit):
```

### Enter the query prompt that you want to query against the Q index
```
# ./data-accessor-tester.sh
Enter your prompt (or 'exit' to quit): find out the status of project x
```

### Authenticate against IAM IDC + IDP from your browser as prompted and provide the authorization code


```
=== AWS OIDC Authentication ===

Please follow these steps:
------------------------
1. Copy and paste this URL in your browser:

https://oidc.us-east-1.amazonaws.com/authorize?response_type=code&client_id=******&redirect_uri=******&state=******

2. Complete the authentication process in your browser
3. After authentication, you will be redirected to: <your redirect url>
4. From the redirect URL, copy the 'code' parameter value

Enter the authorization code from the redirect URL:
```

### The script goes through the rest of proper authentication flow and calls Search Relevant Content API to retrieve the Q index information that matched against your query

```
Calling SearchRelevantContent API...
SRC API Response (High/Very High confidence only)
=================
{
  "relevantContent": [
    {
      "content": "\nProject X Status Report - RED Overall Status: RED  Key Issues:  1. Schedule: Project is currently 3 weeks behind critical milestones............",
      "documentId": "s3://xxxxxx/Project X Status Report.docx",
      "documentTitle": "Project X Status Report.docx",
      "documentUri": "https://xxxxxx.s3.amazonaws.com/Project%20X%20Status%20Report.docx",
      "documentAttributes": [
        {
          "name": "_source_uri",
          "value": {
            "stringValue": "https://xxxxxx.s3.amazonaws.com/Project%20X%20Status%20Report.docx"
          }
        },
        {
          "name": "_data_source_id",
          "value": {
            "stringValue": "xxxxxxx"
          }
        }
      ],
      "scoreAttributes": {
        "scoreConfidence": "VERY_HIGH"
      }
    },
    ......
```

### Final section of the script calls Amazon Bedrock to summarize the Q index data with the query 

```
Summarizing results with Amazon Bedrock (model - amazon.nova-pro-v1:0)...
Calling Bedrock API...
Summary
=================
**Summary for the search query "project x":**

Project X is currently facing significant challenges as indicated by two status reports:

1. **RED Status Report** (Source [1]):
.............

**URI Links:**
- RED Status Report: https://*******.s3.amazonaws.com/Project%20X%20Status%20Report.docx
```

## Clean Up

To remove the solution from your account, please follow these steps:

1. Remove data accessor
    - Go to the AWS Management Console, navigate to Amazon Q Business >  Data accessors
    - Select your data accessor and click 'Delete'
