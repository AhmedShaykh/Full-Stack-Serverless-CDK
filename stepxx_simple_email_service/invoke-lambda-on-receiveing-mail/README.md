# Calling Lambda When Receiveing Emails On Verified Domain

In this example of CDK deployment we will take a look at [how to receive emails on Amazon email server](https://www.youtube.com/watch?v=2fWj3EKYalg&feature=youtu.be&t=735). For a complete list of Regions where email receiving is supported, see [Amazon Simple Email Service endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/ses.html) in the AWS General Reference. Currently AWS is supporting email receiving support in three regions **us-east-1**, **us-west-2** and **eu-west-1**. To receive an email through SES we have to consider the following things:

- We should have our own domain.
- Domain should be verified on Amazon SES.
- We should have atleast one rule define to receive email on given verified email addresses.

If you have your own domain than learn [how to verify your domain on AWS SES](https://www.youtube.com/watch?v=j8izLCTBIwg) and follow the steps below to [create a rule set to receive emails](https://youtu.be/nxXIpPZzMd0).

<br>

### Step 1: Setup a CDK directory
`cdk init app --language typescript`

<br>

### Step2: Install The Following Dependencies
`npm install @aws-cdk/aws-lambda @aws-cdk/aws-ses @aws-cdk/aws-ses-actions`

<br>

### Step3: Setup Your CDK Stack
To receive mails in SES we need to define a rule set first and in each rule set you can add many rules. You can create many rule sets but you can only activate one rule set at a time. Inside a rule you have two main options `recipients` and `actions`. inside `recipients` you have to define an email address on which you want to recieve emails and inside `actions` you can add some resource to take action when receive an email on your verified address. For example if you add an email address like **info@example.com** in your rule, so you can only get response when you receive email on this address **info@example.com** which is pretty simple to do with CDK. If you recieved any email on an address which is not available inside your rule than that email will be deleted.

```javascript  
// code to create a rule set
import * as ses from '@aws-cdk/aws-ses';

const ruleSet = new ses.ReceiptRuleSet(this, 'RuleSet', {
      receiptRuleSetName: 'my-rule-set',
    })
```

<br>

In this example we will use lambda as an action when receive email on verified email address. Add the code below inside your **lib/stact.ts**.

```javascript
// lib/stack.ts
import * as cdk from '@aws-cdk/core';
import * as lambda from '@aws-cdk/aws-lambda';
import * as actions from '@aws-cdk/aws-ses-actions';
import * as ses from '@aws-cdk/aws-ses';

export class Stack extends cdk.Stack {
  constructor(scope: cdk.Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // The code that defines your stack goes here

    // Setting a lambda function which will be called on receiveing email
    const actionLambda = new lambda.Function(this, 'SES_ACTION_LAMBDA', {
      code: lambda.Code.fromInline(
        `exports.handler = (event)=>{ console.log("EVENT ==>> ",JSON.stringify(event)) }`
      ),
      runtime: lambda.Runtime.NODEJS_12_X,
      handler: 'index.handler',
    })

    // creating a new rule set
    const ruleSet = new ses.ReceiptRuleSet(this, 'RuleSet', {
      receiptRuleSetName: 'calling-lambda-rule-set',
    })

    // creating instance for taking email input when deploying stack
    const emailAddress = new cdk.CfnParameter(this, 'emailParam', {
      type: 'String', description: "Write your recipient email: "
    });

    // Adding a rule inside a rule set
    ruleSet.addRule('INVOKE_LAMBDA_RULE', {
      recipients: [emailAddress.valueAsString], // if no recipients than the action will be called on any incoming mail addresses of verified domains
      actions: [
        new actions.Lambda({ // defining an action to call when receive email on given recipients
          function: actionLambda,
          invocationType: actions.LambdaInvocationType.EVENT,
        }),
      ],
      scanEnabled: true, // Enable spam and virus scanning
    })

  }
}

```

<br>

### Step4: Lets deploy
After completing the above steps run the following commands to deploy
- `npm run build`
- `cdk deploy --parameters emailParam=info@example.com`
> Note: your have to replace **info@example.com** with your verified email address

<br>

### Step5: Test Your Rule
Now its time to test that your lambda should be called when you received an email on **info@example.com**. To test it you can use any of the free mailing service like **Gmail**, **Yahoo** or any one you like and send an email to **info@example.com** from your email address. 

Once you have send the email, you can now go to your lambda console and check the log inside **CloudWatch** logs.
![](images/img1.JPG)