# SenNet AWS API Gateway

## Create REST API in AWS API Gateway via import

In AWS API Gateway, a REST API refers to a collection of resources and methods that can be invoked through HTTPS endpoints. The easiest way of creating a new REST API is by importing the OpenAPI v3 specification yaml file.

Note: a valid OpenAPI v3 yaml file that renders correctly in Swagger or SmartAPI editor does not guarantee it's valid to the AWS API Gateway import tool, modifications may be necessary.

Once a REST API is finialized, we can deploy it to a certain stage and export the definition details as "OpenAPI 3 + API Gateway Extensions" yaml, which contians all the integration details and authorizors defined to be used by the resources. This can be used to restore the REST API when needed. So it's a good practice that we always export this yaml for each API PROD release.

## APIs running on AWS

### Overview of request route

```
Client -> Custom domian name -> AWS API Gateway target stage -> Lambda Authorizer -> 
NLB of the target stage via Proxy integration (VPC Link) -> Target Group and TCP port -> REST API endpoint
```

### General implementation workflow

- Create a Target group per REST API for each deployment stage (DEV/TEST/STAGE/PROD), define a unique TCP port number to be used to communicate to the EC2 instance for each API (currently we use TCP port 2222 for uuid-api, 3333 for entity-api, and 4444 for search-api)
- Create an internal Network Load Balancer (NLB) with mappings to all the VPC Availability Zones for each deployment stage and specify TCP listeners for each Target group and the corresponding ports.
- Create a security group for each NLB, and add the primary private IPv4 of each Availability Zone (can be found under "Network interfaces" of EC2 console)
- Attach the security group to the target group's EC2 instance so the NLB is allowed to access the target EC2 instance on those defined ports
- Create a "VPC Link for REST APIs" for each deployment stage and link to the corresponding NLB
- Import target API's openapi specification yaml file to AWS API Gateway
- Enable CORS via OPTIONS method for each resource instead of using API Gateway's CORS option
- Choose VPC Link integration for each resource's method, check "Use Proxy Integration" and choose "Use stage variables" with value of `${stageVariables.VPCLINK}`, also use stage variable to define Endpoint URL, example:
`http://${stageVariables.VPCNLB}/ancestors/{id}`
- For each deployed stage, set the two stage variables: `VPCLINK` (the generated ID of the VPC Link created earlier) and `VPCNLB` (the DNS of NLB with the target group port, e.g., `NLB-STAGE-4fc5be9e0b9f2bd6.elb.us-east-1.amazonaws.com:3333`)
- Request a new ACM public certificate (choose DNS validation) for each REST API of each deploy stage via AMC console (to be used by custom domains) and click "Create records in  Route53" record from the ACM console when the certificate DNS validation status is pending
- Create API Gateway custom domain name of each API on each stage and select the existing ACM certificates, take note of the "API gateway domain name" value for later use
- Create the Route53 DNS record for each domain, use "API Gateway domain name" (generated by each API Gateway custom domain under the Endpoint configuration section) as the CNAME value
- Configure "API mappings" for each API Gateway custom domain name to map to the corresponding API stage

## APIs running on PSC

Very similar to APIs running on AWS descripted before, the APIs running on PSC VMs are proxied through the Nginx of the "Old" Gateway on AWS to the dedicated ports on each corresponding PSC VM.

```
Client -> Custom domian name -> AWS API Gateway target stage -> Lambda Authorizer -> 
NLB of the target stage via Proxy integration (VPC Link) -> Target Group and TCP port -> Nginx reverse proxy on EC2 instance of each stage -> PSC Firewall -> REST API endpoint
```

- Ingest API on port 8443

## How to create lambda function dependencies layer

According to https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html we need to define the same folder structure in our layer .zip file archive as the ones supported: `python` or `python/lib/python3.9/site-packages`. The easies way is to just install all the dependencies to the `python` directory under this project root directory then zip it as `python.zip`:

```
pip install -r requirements.txt --target python
zip -r python.zip python
```

Next, in AWS console create a new layer in AWS Lambda and upload this zip archive with selecting `x86_64 architecture` and `Python3.9` runtime (the dependencies are installed under Python3.9.2 on my local), and then add this custom layer to each of the authorizer lambda functions.

When the dependencies get updated, recreate the .zip archive and upload to the AWS lambda layer as a new version. Will also need to specify this new layer version for each lambda function that uses it.

## Custom 401 and 403 response template

The variable `$context.authorizer.key` is made available in the authorizer lambda function to send back more detailed information. And In Gateway Responses pane, we use the following template to transform the body before returning to the client for only 401 and 403 responses:

```
{
    "message": "$context.error.message",
    "hint": "$context.authorizer.key",
    "http_method": "$context.httpMethod"
}
```

Note: when the `Authorization` header is not present from the request, it seems AWS API Gateway just returns 401 with the `$context.error.message` being "Unauthorized" and the authorizer lambda function never gets called. Thus why `$context.authorizer.key` is not set.

## Handle undefiend resources with 404

By default AWS API Gateway returns 403 response on any undefined endpoints instead of 404. To solve this issue, we need to
- add an `ANY` method to the root API
- add a `/{proxy+}/ANY` method to the root API
- use Lambda Proxy integration with the `404.py` lambda function which simply returning a 404 response on any undefiend endpoints

Then the 404 response will be returned to undefined resurces or undefiend method on the resource:

```
{
    "message": "Unable to find the requested resource"
}
```

## API redeployment

Each API stage created in API Gateway can be linked to a specific deployment, this also allows us to roll back to an earlier deployment. To deploy new API changes:

- update the resource definition and deploy this new version to the target stage (if needed) in API Gateway
- deploy the backend API changes to the actual EC2 instance

## Access to non-PROD APIs

AWS WAF (Web Application Firewall) is integrated to monitor the HTTP and HTTPS requests that are forwarded to the API Gateway REST APIs. Below is the workflow:

- Create a new WAF web ACL
- Associate the Web ACL to each API and its target deployment non-PROD stage 
- Create a new IP Set with the IP addresses that are allowed to access those non-PROD APIs
- Add the IP Set as one of the Rules of the previously created Web ACL
- Create a custom response JSON body
- Set the default Web ACL action to BLOCK requests that don't match any rules and return 403 status with the custom response body

Custom response body:
```
{
    "message": "Sorry, your IP is blocked. Please Contact help@sennetconsortium.org to request developer access to the target resources."
}
```
