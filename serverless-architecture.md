# Serverless architecture on AWS

N-tier architecture components - network, database, MVC application, REST endpoints, application/web servers

- Utilize VPC, OpsWorks (AWS version of Chef), Cloudformation (AWS IaaS automation scripting), EC2, RDS/DynamoDB, S3, API Gateway, AWS Lambda and Route 53 to create equivalent N-Tier architecture.
- For serverless apps,the architecture is hence translated as :

<ol>
<li> DynamoDB for data </li>
<li> AWS Lambda to act as DAO </li>
<li> API Gateway to act as integration layer exposing the endpoints for invoking the lambda </li>
<li> S3 acts as the presentation layer, and also doubles up as static webstite hosting and providing object storage </li>
<li> Secure the architecture using VPC by restricting access to the provisioned infrastructure along with WAF and SSL </li>
<li> Route 53 can be used to provision a DNS for the app </li>
</li>


### Demo - Backend DynamoDB

- Create a simple table in DynamoDB and create a table. Minimum details.

Table Name : Books
Primary Key : bookId
Columns : bookId, author, bookName

Once table is created, go to Items tab and create a table.

### Demo - DAO - AWS Lambda

- Use a simple template with Node.js backend and create a function named "Book Details"
- Use the dynamoDB Client to query the books table, add callback and log errors to console.

### Demo - Integration Layer - API Gateway

- Create a GET Endpoint invoking the existing lambda function (the "Book Details" function created in previous step). Use the same region as Lambda.
- In the body mapping template, specifiy Content-Type = 'application/json'
- Mention method body as "Passthrough" so everything from the request including header, query params and body is available to the lambda.
- Test and verify the API gateway can invoke the lambda and return a HTTP 200.
- Actions -> Deploy -> Deployment -> Stage, and give a description. Create a new environment where we want to promote this endpoint.
- You can invoke this API from the endpoint where the API Gateway is deployed.
- Use the SDK -> Generate Client -> Javascript option to generate a REST client. This can be used to invoke the app from S3.

### Demo - Presentation Layer - S3

- Create a simple HTML form to query the Books table by bookId. Upload this HTML, the generated SDK client for api gateway and generated lib directory
to a new S3 bucket. Name should be globally unique and if exposed on Route 53, should match the DNS hostname.
- Set CORS on the API Gateway. Make resources public access, so it can be accessed as a static site. Redeploy the gateway.
- Form can be invoked using the autogenerated endpoint provided for the S3 static site hosting.
- When then the form submit the SDK client invokes the lambda and returns the value of data from Book Details DynamoDB collection.

### Demo - Assign Domain - Route 53

- Pre-requisites : A working domain name pre-configured in Route 53 with NS, SOA and A records.
- Create a new Alias record set with following details :
```
Name : books.abc.com
Type : A - IPV4 Address
Alias Target : The S3 bucket endpoint
```
- Now try to invoke using domain name from your browser. It should redirect to the static site set up in S3.
- Hence we are able to create a MVC serverless app with a UI, DAO, database, and REST endpoint.

### Security Considerations

#### API Gateway Security Considerations :
- Design API properly and protect them using IAM policies.
- Secure your public facing API with API Keys (Least recommended, know the risks and use them properly).
- Allow CORS only for recognized domains.
- The correct way to secure API Gateways is by using custom authorizers (SAML, OAuth, Custom Tokens).
- protect your backend by using and validating client SSL Certificates.

## Serverless Application Security

### Application Security using Cognito User pool

#### How to create user pool
- Cognito -> Manage User Pools -> Create User Pool -> Demo_User_Pool -> Step through settings (so we can review them).

* Which standard attributes you want to require?
- Email (So users can login with email address)

* Password Strength - Choose min length, require attributes like numbers, special chars, upper case and so on.

* Do you want to allow users to sign themselves up? Yes, sign themselves up

* Do you want to enable Multi-Factor Authentication (MFA) ? No

* Do you require email / phone number verification ? Email verification

* Provide a role for cognito to send SMS messages (Create Role)

* Customize Verification Messages? Use default templates. 

* Do you want to add tags to user pool? Add an AppCode called "DEMO"

* Do you want to remember user devices? NO

* Will application clients have access to user pool ? If the client is going to access the pool programatically, they need to provide a unique ID and client secret to sign in. Our Demo app requires this . Create this App client.

* Do you want to customize workflows with triggers? No

Once User pool is created, we can edit users and groups.
 - Create User, send invite to user using email. Set a temporary password, and a dummy email address (user1@mailinator.com). User is enabled. 
 Credentials valid only one time, user will be asked to change password.
 
 #### Cognito User IDP sign up
 
 ** Step 1 **
 
 * Install AWS CLI and then invoke the pool from command line.
 
 
** Step 2 : Get the Session for User pool **
 
 ```
 aws cognito-idp admin-initiate-auth --user-pool-id us-east-2_sdfsfsdfds --client-id sdfsdf234 --auth-flow ADMIN_NO_SRP_AUTH --auth-parameters USERNAME=user1, PASSWORD=mypass --region=us-east-2
 ```
 
Cognito will return the session state in response. Next request will use the session state.

** Step 3 : Change Password ** 

 ```
 aws cognito-idp admin-respond-to-auth-challenge --user-pool-id us-east-2_sdfsfsdfds --region us-east-2 --client-id sdfsdf234 --challenge-name NEW_PASSWORD_REQUIRED --challenge-responses NEW_PASSWORD=password, USERNAME=user1 --session
 ```
 
 When above query is fired, Cognito returs a JWT response with a bearer token with Access Token, Refresh Token.
 
 ** Step 4: User Sign up **
 
 ```
 aws cognito-idp sign-up --client-id 345sddre --username user2 --password pass --user-attributes Name=email, Value=user2@mailinator.com --region us-east-2
 ```
 
 Once user is added, default status is unconfirmed. Access the mailinator inbox and verify the email.
 
 ** Step 5: Confirm the sign up **
 
 ```
 aws cognito-idp confirm-sign-up --client-id sdfdfsfsdf --username user2 --confirmation-code 12345 --region us-east-2
 ```
 
 Firing this command should confirm the user. Now in AWS console, we can see user status is confirmed.
 
  #### Deploying Demo application
  
  - Use the demo application we uploaded to S3 in previous step. If you have not yet uploaded the app to S3, you can do it via AWS CLI.
  
  ```
  aws s3 cp ~/Desktop/app s3://bookstore.myapp/ --recursive --acl public-read --region us-east-2
  ```
  
  - Access the app from S3 endpoint to see if it works upto DynamoDB (testing minus Route 53).
  - Deploy API to the same stage if needed or if any errors are fixed (enabling CORS).
  
  #### Securing API Gateway using the user pool
  
  - Go to API Gateway and access the endpoint we just created
  - Using the "Authorizers" tab, create a new custom authorizer.
  - New Custom Authorizer -> Create -> New Cognito User Pool Authorizer -> Region = us-east-2, DemoUserPool, method.request.header.Authorization -> Update
  - Resources tab under the API -> /bookdao -> POST -> Method Execution -> Method request -> Authorizaton -> Demo_User_pool -> Re-deploy the API to reflect the change.
  
 Now if we test from REST Client, we will get 401 Unauthorized, hence API is secured only for users in Cognito User pool.
 
- Use admin-initiate-auth command from above, give user name, password and get the tokens.
- Use the "IdToken" from the cognito Auth response and add a "Authorization" header and add this token on the header.
- Upon entering the right Token, we can now invoke the endpoint from a rest client (Eg. REST Client / Postman), and get a HTTP 200.

  
#### Update Demo App to include authentication 
- Add Cognito client to the generated javascript client
- Retrieve the token header for "IDToken" and pass it to the REST API as a "Authorization" header to invoke the API Gateway endpoint.








