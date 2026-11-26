# Cloud Café: an event-driven ordering system on AWS

Final lab of my AWS Cloud Architecting course. Customers place an order,
it is stored, cooked by a pool of workers, and analyzed, with a disaster
recovery plan behind it.

## Architecture
![Architecture](architecture/cloud-cafe-architecture.png)

## What I built
| Layer | Services | Purpose |
|---|---|---|
| Infrastructure as code | CloudFormation | One template builds the whole stack |
| API | API Gateway (HTTP API), Lambda | Place and read orders |
| Storage | DynamoDB, S3 | Orders table and a data lake |
| Messaging | SNS, SQS, dead-letter queue | Fan-out and a buffer for the kitchen |
| Compute | EC2 Auto Scaling group | Kitchen workers that poll SQS |
| Workflow | Step Functions | Order check, no Lambda |
| Analytics | Glue, Athena | SQL on the data lake |
| Monitoring | CloudWatch alarms, SNS | Backlog and DLQ alerts |
| Security | IAM roles, security groups | Least privilege |

## How a request flows
1. The client calls API Gateway (POST /api/orders).
2. Lambda saves the order in DynamoDB, publishes it to SNS, and writes a JSON file to S3.
3. SNS sends a copy to the SQS kitchen queue (and an email for orders of $20 or more).
4. Workers in the Auto Scaling group long-poll SQS, cook the order, and update DynamoDB through a gateway VPC endpoint.
5. A message that fails 3 times moves to the dead-letter queue and triggers an alarm.

## Security
- Workers use an IAM role (no access keys) and a security group with no inbound rules.
- A restricted IAM user can only read order files, and can never delete (explicit deny).
- The S3 buckets are private and encrypted.

## Disaster recovery
- DynamoDB on-demand backup and restore. Restore time: [X] minutes.
- S3 cross-Region replication of the data lake to eu-west-1.
- Game day: I terminated the worker on purpose. Orders waited in the queue and Auto Scaling launched a replacement. RTO [X] min, RPO [Y].

## Troubleshooting notes
- CloudFront could not be created ("account must be verified"), so I deployed with CloudFront off.
- "Failed to fetch": opening the page from file:// sends origin "null", which the browser blocks (CORS). Serving it from a local web server fixed it. I also learned that CORS is configured on API Gateway, not in the Lambda code.

## Cost control
Zero-spend budget, and a full cleanup at the end. DynamoDB backups and the DR bucket are not deleted by the stack, so I removed them manually.

## What I would add for production
- Private subnets with a NAT gateway or VPC endpoints
- CloudFront + WAF with HTTPS
- KMS keys and Secrets Manager
- Route 53 failover and a warm standby Region

## Screenshots
![Stack created](screenshots/01-stack-create-complete.png)
![Step Functions](screenshots/08-step-functions-succeeded.png)
![Athena](screenshots/09-athena-best-sellers.png)

## Cleanup proof
![Stack deleted](screenshots/11-cleanup-delete-complete.png)
