# Cross-tenant subscriber engagement emails in Power Automate
>Integrating Dataverse & SharePoint with batched HTTP requests in Power Automate for cross-tenant, automated email campaigns.

## Contents

## Overview
This project automates subscriber engagement emails using two Power Automate flows across separate Microsoft tenants. Flow A runs daily, checks a SharePoint list where email content and audience details are stored, and constructs dynamic OData filter queries to extract subscriber data from Dataverse (Dynamics 365 CRM). Finally, Flow A sends a single HTTP request to Flow B, triggering it. Flow B parses the POST request and sends the emails from its tenant.

## Use Case
Tenant A contains all subscriber data (D365 CRM), a SharePoint list - already used for scheduling automated emails, and a Power Automate license - again, already used in various automated processes including sending regular engagement emails. Owing to a recent platform update and rebranding of a core product line, a separate MS tenant (Tenant B) was created, containing licensed and shared mailboxes and a Power Automate Premium licensed user.

**The business problem:**

- Subscriber engagement emails must be sent using Tenant B's mailboxes
- All subscriber data and email content / metadata is stored in Tenant A
- Power Automate does not allow users to simply "plug in" connectors from outside your own tenant

**The fix:**

- Extract and transform all the data needed for the email campaign, including:
	- Subscriber contacts and names (filtering by subscription type and job role)
	- Email content and metadata
	- Base-64-encoded inline image and attachment content
- Pass as a single batched HTTP payload to Flow B, and send the email

![image](https://github.com/user-attachments/assets/077969f1-26c6-4c2c-b8d2-22219d0b4e70)
>Process diagram

## How it works

### Flow A
>Get data, send HTTP request

For the purposes of this guide, I have skipped over various intermediary steps to do with lookups across various Dataverse tables. This is to do with the specific design of the organisation's CRM, and is not necessarily generalisable to the task at hand. I have, however included details on constructing dynamic filter queries; precisely how these will work in your use case will depend on the structure of your own organisation's database / CRM system.

#### Trigger
Flow A runs on a recurrence trigger each morning. There was no requirement for emails to be sent at a specific time, so I just picked 1200. If there were such a requirement, `time_of_email` could be stored as an additional column in the SharePoint list.

#### SharePoint List
Users wishing to schedule emails can do so by updating a SharePoint list. Email attachments can also be added per row.

![image](https://github.com/user-attachments/assets/3e198fd8-0990-444b-bb04-71396d46d522)
>A row in the SharePoint list containing email content, and details of who it should be sent to (in this case: subscribers with `Membership type = Training OR conferences` and `Audience = Senior Leadership Team`).

#### Check if emails scheduled for today
In Power Automate, using the SharePoint *Get Items* action, the flow gets emails form the list which scheduled for today, filtering by `Turn on = True`.

```JSON
"$filter": "Turnon eq 1 and Sendon eq '@{formatDateTime(utcNow(), 'yyyy-MM-dd')}'"
```

![image](https://github.com/user-attachments/assets/5524bdf4-ef89-4ab5-8da4-e75d8a42b34f)

A condition action checks if the length of the output of *Get items* is greater than 0. If false, the flow terminates (there are no emails scheduled today). If true, the flow continues.
#### Get inline image content
Each email includes an image in the signature (the company logo). Use the *Get file content* action to access the .png file content from SharePoint (assuming the image is stored in SharePoint). The output from this action will be used to compose the email content in a later step.

#### Loop over each email
For each row in the output of `Get items`, iterate over them so the below steps are repeated for each scheduled email.

![image](https://github.com/user-attachments/assets/9347275e-1be4-47b4-b456-b85ddecf8d3f)

#### Construct dynamic OData filter queries
Here comes the tricky part. For each scheduled email, the audience type can differ by subscription type and job role. There is no way to know in advance what combination of subscription type and job role will be required.

>	As mentioned, for the purposes of this guide, I will focus only on filtering by job role (and skip filtering by `membership type`) as this step is more likely to generalise to other use cases. In the organisation's CRM, subscription data is stored across multiple tables at the `account`, `contact` and `product` level, the details of which are not relevant to our general goal. Suffice it to say: the process is largely the same minus some additional lookups, transformations and joining of tables.

##### Store the items from the `audience` column in an array variable
Loop over each item of the `Audience` column in your SharePoint list, and append to an array variable. In my flow, the variable is called `audience`.

![image](https://github.com/user-attachments/assets/4c770695-b8f1-4128-871e-fc20a2b4307b)

If the response contains newlines, which is often the case in Power Automate, use this string operation in the append action to strip them out.

```plaintext
replace(items('Apply_to_each_audience')?['Value'], '\n', '')
```

**Note** - before the `Apply to each` loop, reset your variable as a blank array. If you don't, the variable will keep getting updated with all job roles for all scheduled emails, instead of the job roles for the current row in the SharePoint list. Remember, we are [[#Loop over each email|currently iterating over]] all of today's emails.

**Also note** - You must initialise the variable at the start of the flow. Power automate does not allow you to initialise variables inside a loop.

##### Use condition actions to construct another array variable
The next step is to map the items in your `audience` variable onto another array, which I've called `send_to`. In my case, each `audience` in the SharePoint list could correspond to several different job roles in CRM. If - in your use case - there is a 1-2-1 relationship between `audience` and `job role`, this step can be skipped.

![image](https://github.com/user-attachments/assets/362819ef-054a-4139-916b-f226b4791d6f)
This condition checks whether the `audience` array, created in the previous step contains a given string, in this case "SLT" (short for "Senior Leadership Team"). If true, we append the corresponding job roles in CRM to another array variable, for example ("Director", "CEO", "Head of Operations"). *More precisely, the numerical Dataverse option set value is appended.*

Chain condition actions together checking whether the given string is contained in the `audience` variable. 

This approach is slightly long-winded, and a more elegant approach might be to store the job role values in an object array and append them in bulk. I favoured my approach - firstly - because the mapping can get very fiddly in Power Automate's UI, and - secondly - because it's more modular and easier to edit in the case that the CRM option set (or the definition of which job roles correspond to which audience) is changed.

##### Use a Compose action to create the $filter query
Concatenate the items in your `send_to` array using this operation in a Compose action. (Replace `jobrole` with your actual Dataverse column name.)

```plaintext
concat('(jobrole eq ', join(variables('send_to'), ' or jobrole eq '), ')')
```

The output of the Compose action will look something like this:

```plaintext
jobrole eq 12345 or jobrole eq 23456 or jobrole eq 34567
```

In my example, the values `12345`,`23456`, and `34567` each correspond to "Director", "CEO" and "Head of Operations" in the Dataverse option set.

##### Pass dynamic filter query in List Rows action
Now you have constructed your filter query, you can use this on its own or in conjunction with hard-coded queries.

![image](https://github.com/user-attachments/assets/4858c0e2-905d-4eb4-be03-ae9335fd10b5)
>Filtering by `status` and `statusreason` are active and the string generated in the previous step.

##### Store emails and names as an array of objects
Once you have listed your subscribers, using the Dataverse List Rows action and the dynamic filter, store each subscriber as an array of objects by iterating over each row in the output of List Rows and appending to an array variable, which I've called `names_with_emails`.

```plaintext
{
  "name": @{items('For_each')?['firstname']},
  "email": @{items('For_each')?['email']}
}
```
>Append the `name`, `email` and any other columns you need to an array of objects inside an Apply to Each loop, iterating over the output of List Rows

#### Get attachments
If your SharePoint list has attachments, use the Get Attachments action.

![image](https://github.com/user-attachments/assets/64992ac0-fee7-4b51-a6a6-41f687ecc56e)
>Declare `attachment_array` as a blank variable at the start of the process. Remember to reset it outside of the Apply to each loop, otherwise it will keep getting updated with each email's attachments.

For each attachment, append the following values to `attachment_array` using the following structure:
^attachmentarraysyntax
```plaintext
{
  "Name": "@{item()?['DisplayName']}",
  "ContentBytes": {
    "$content": "@{body('Get_attachment_content')?['$content']}",
    "$content-type": "@{body('Get_attachment_content')?['$content-type']}"
  }
}
```
>`DisplayName` is part of the output of Get Attachments. `content` and `content-type` are from Get Attachment Content

#### Make HTTP request
Use the Make an HTTP request action with the following configuration.

![image](https://github.com/user-attachments/assets/c720e0fd-d562-4ccd-acd8-c522676ba506)
>- Method : POST
>- Content-Type : `application/json`
>- Body:
>	- `"to"` : your [[#Store emails and names as an array of objects|array of objects]] containing subscriber names and emails
>	- `"subject"` : the email subject line, passed directly from the SharePoint list
>	- `"body"` : the email content, passed directly from the SharePoint list
>	- `"attachment_array"` : the array variable created in the [[#Get attachments|last step]] , containing attachment metadata and base-64 encoded content
>	- `"image_inline"` : the base-64 encoded content of your inline image, [[#Get inline image content|passed directly from a SharePoint file]]
>	- `"fromFlowName"` : *optional* - the name of the flow sending the request, for tracking purposes

## Flow B
>Receive HTTP request, send email

### Trigger
Use the When an HTTP request is received trigger (requires Power Automate Premium license). Define the schema in the following way:

```JSON
{

    "type": "object",

    "properties": {

        "to": {

            "type": "array"

        },

        "subject": {

            "type": "string"

        },

        "body": {

            "type": "string"

        },

        "attachment_array": {

            "type": "array"

        },

        "image_inline": {

            "type": "string"

        },

        "fromFlowName": {

            "type": "string"

        }

    },

    "required": [

        "to",

        "subject",

        "body"

    ]

}
```

### Send email
Use an "Apply to each" loop, iterating over your `to` (which stores your [[#Store emails and names as an array of objects|`names_with_emails`]] array of objects). Use the following dynamic content in the Outlook Send an email (V2) action.

```JSON
"parameters": {

      "emailMessage/To": "@items('Apply_to_each_1')['email']",

      "emailMessage/Subject": "@triggerBody()?['subject']",
```

Depending on how your email is laid out, you can reference the first name of your subscriber in the first line, before passing the `body` parameter as the main content.

``` JSON
"emailMessage/Body": "<p>Dear @{items('Apply_to_each_1')['name']},</p><p>@{triggerBody()?['body']}</p>
```

#### Add email signature image inline
Ensuring you are using the edit HTML mode, add the following code to add your inline image, adjusting the size as required.

```HTML
<img src=\"data:image/png;base64,@{triggerBody()?['image_inline']}\" width=\"412\" height=\"93\">
```

#### Add attachments
The hard work is already done! Simply pass your `attachments_array` in the Send an email (V2) action.

![image](https://github.com/user-attachments/assets/8b1ab824-6fc3-4608-b2eb-deeacb34ebfc)

>**Note**: If you find your attachments are corrupted or not attached when you receive the email, it is likely due to a malformed array. Ensure that the syntax is as specified in the [[#^attachmentarraysyntax|previous step]].

## What I learned
- Within reason, Power Automate will do anything you want it to **if** you force it to. It's deceptively flexible at times; dynamic OData queries are a powerful tool, but they get messy fast without proper planning.
- This was a fun use case for a low-code / no-code tool. Having said this, it's funny how "low-code" often translates to "debugging JSON at midnight" when your process logic is complex enough. Power Automate's UI can be frustrating at times, in that it allows you to peek the code, but altering it isn't always intuitive or possible. At times, I really felt this project would have gone a lot faster if I'd built it entirely in code.
- Chaining condition blocks together to create the dynamic filter for `audience` worked in my use case, but it's not very scalable. If I were to build this again, I would look into more sophisticated mapping.
- Corrupted attachments are the worst. Save yourself two days: use [[#^attachmentarraysyntax|this syntax.]]

## Contact
Written by Mike Brennan

Connect with me on [LinkedIn]([(1) Michael Brennan | LinkedIn](https://www.linkedin.com/in/michael-brennan-22312014a/))
