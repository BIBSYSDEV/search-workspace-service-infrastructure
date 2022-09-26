## Search Workspace Service (SWS) Infrastructure deployment 

## Preliminaries:

In this document:

1. We refer to a production or non-production domain as the CustomDomain .  
   We refer to the value of the CustomDomain  as `$CustomDomain` (e.g. `dev.sws.aws.sikt.no` or `sws.sikt.no`)/.  
   For example if `$CustomDomain = "dev.sws.aws.sikt.no"` ,  then `api.$CustomDomain = "api.dev.sws.aws.sikt.no"`

2. If Region is not explicitly mentioned we are always in Region **eu-west-1**.

3. All AWS Service names appear in italics e.g. (_CloudFront_).

<a href="#Assumptions"></a>
## Assumptions:
1. The production domain is `sws.sikt.no` .
2. The non-production domains have the form `<env>.sws.aws.sikt.no` where `<env>` is typically "dev", "test", "sandbox".
3. There is a _Route53_ Hosted Zone with the property "Domain name" equal to `$CustomDomain`.
4. There is a Certificate in _Certificate Manager_ in Region **us-east-1**  that has two records for the domains:
    * `$Domain`   (e.g. `dev.sws.aws.sikt.no`)
    * `*.$Domain` (e.g. `*.dev.sws.aws.sikt.no`)

## Deployment

#### Backups:
Take a snapshot of the opensearch-clusters and store it somewhere outside of the stack

### Cleanup

1. Go to _CloudFormation_ -> Stacks, and search for SwsPipeline. Delete the one which is NOT nested
2. After this stack has been deleted (can take 20min), still in _CloudFormation_ search for and delete "sws-master-pipeline"
3. Go to Route 53 and delete the CNAME records for api.<env>.sws.aws.sikt.no pointing to a cloudfront

#### Manual configuration

1. **S3 Bucket containing the SWS-Infrastructure files**:
Assert that there is an S3 bucket containing the latest version of this repository.  
This bucket is maintained **manually**, and you should pull the `master` branch and copy  
the contents of the folder "cloudformation" in `search-workspace-service-infrastucture` repository (as is) to the S3 bucket **manually**.
2. **Certificates:**
Assert that a certificate is in place in _Certificate Manager_ in Region **us-east-1** and it contains the aforementioned records (see [Assumptions](#Assumptions))
3. **DNS (_Route53_):**
   1. In _Route53_ ,locate the HostedZone with Domain name equal to `$Domain`. (e.g `dev.sws.aws.sikt.no`).
   2. In the HostedZone, delete any CNAME records that contain the values `api.$Domain` (e.g., `api.dev.aws.sws.sikt.no`).
   3. In the HostedZone, delete any CNAME records that contain the values `portal.$Domain` or any CNAME record is attached to a developer portal (Open API documentation page).
   4. **ATTENTION:** Do not delete any CNAME records that have "Domain value" equal to `$Domain`.
   5. **ATTENTION:** Do not delete any **non**-CNAME records
   6. **ATTENTION:** The Hosted Zone for production is `sws.aws.sikt.no`.
4. **Secrets:** Set-up the following secrets, if they are not in place:
   1. Githubtoken
                Secret name: githubtoken
                Secret key:   SecretString  
                Secret value: `<githubtoken>`   
                The values for the non-production environments are shared.   
5. **Build Master stack:**
   1. Create a new Master stack using the `pipelines_master_template.yml` from the s3 bucket created earlier.
   2. Fill in the details. The defaults are for dev-environments. 
6. **Update DNS for Backend:** 
   1. Go to _ApiGateway_ --> "Custom domain names" and select the created custom domain name (e.g. `api.sandbox.sws.aws.sikt.no`).
   2. Copy the value of the field `API Gateway domain name` (e.g. `d27gccxh1hqvcd.cloudfront.net`)
   3. Create a new CNAME Record in the associated Hosted Zone in _Route53_, e.g.:

                    Record name: api.sandbox.sws.aws.sikt.no
                    Record type: CNAME
                    Value: d27gccxh1hqvcd.cloudfront.net
