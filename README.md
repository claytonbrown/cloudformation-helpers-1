# cloudformation-helpers
A collection of AWS Lambda funtions that fill in the gaps that existing CloudFormation resources do not cover.

AWS CloudFormation supports Custom Resources (http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html),
which can be used to call AWS Lambda functions. CloudFormation covers much of the AWS API landscape, but
does leave some gaps unsupported. AWS Lambda can contain any sort of logic, including interacting with the
AWS API in ways not covered by CloudFormation. By combining the two, CloudFormation deploys should be able
to approach the full resource support given by the AWS API.

Warning: most of these functions require fairly wide permissions, since they need access to resources in a
general manner - much the same way CloudFormation itself has permission to do almost anything.


## Usage
1. Upload the .zip file of this repo from Github to an S3 bucket in your AWS account.
2. Use the included create_functions.template to deploy a stack that creates the Lambda functions for you. Remember the stack name.
3. Include the following resources in your CloudFormation template. These will create a) a nested stack that
   looks up the ARNs from the previous step and b) a custom resource that allows your template to read those ARNs.
   
   ```
   "CFHelperStack": {
     "Type": "AWS::CloudFormation::Stack",
     "Properties": {
       "TemplateURL": "https://s3.amazonaws.com/com.gilt.public.backoffice/cloudformation_templates/lookup_stack_outputs.template"
     }
   },
   "CFHelper": {
     "Type": "Custom::CFHelper",
     "Properties": {
       "ServiceToken": { "Fn::GetAtt" : ["CFHelperStack", "Outputs.LookupStackOutputsArn"] },
       "StackName": "your-helper-stack-name-here"
     },
     "DependsOn": [
       "CFHelperStack"
     ]
   }
   ```
   
   You can either hardcode the stack name of your helper functions, or request it as a parameter.
4. Use the ARNs from the previous step in a custom resource, to call those Lambda functions:

   ```
   "PopulateTable": {
     "Type": "Custom::PopulateTable",
     "Properties": {
       "ServiceToken": { "Fn::GetAtt" : ["CFHelper", "DynamoDBPutItemsFunctionArn"] },
       "TableName": "your-table-name",
       "Items": [
         {
           "key": "foo1",
           "value": {
             "bar": 1.5,
             "baz": "qwerty"
           }
         },
         {
           "key": "foo2",
           "value": false
         }
       ]
     },
     "DependsOn": [
       "CFHelper"
     ]
   }
   ```


## Included functions

### Insert items into DynamoDB

Pass in a list of items to be inserted into a DynamoDB table. This is useful to provide a template for the
content of the table, or to populate a config table. There is no data-checking, so it is up to the client
to ensure that the format of the data is correct.

Warning: it is a PUT, so it will overwrite any items that already exist for the table's primary key.

#### Parameters

##### TableName
The name of the DynamoDB table to insert into. Must exist at the time of the insert, i.e. will not create if
it does not already exist.

##### Items
A JSON array of items to be inserted, in JSON format (not DynamoDB format).

#### Reference Output Name
DynamoDBPutItemsFunctionArn