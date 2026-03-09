# IAM Activity Alert CloudFormation Template

This CloudFormation template sets up automated alerts for IAM user creation activities in your AWS account. It monitors `CreateUser`, `CreateAccessKey`, and `CreateLoginProfile` events and sends notifications via email (SNS) and/or Slack.

## Features

- **EventBridge Rule** monitoring IAM events via CloudTrail
- **Flexible Notifications**: Choose email, Slack, or both
- **Customizable User Exclusions**: Exclude service accounts from alerts
- **Rich Slack Messages**: Color-coded messages with detailed event information
- **Email Notifications**: Formatted email alerts via SNS
- **Easy to Share**: Parameterized for easy deployment across multiple AWS accounts

## Prerequisites

1. **CloudTrail must be enabled** in your AWS account (management events logging)
2. For Slack notifications: Create a Slack Incoming Webhook URL
   - Go to https://api.slack.com/messaging/webhooks
   - Create a new app or use existing one
   - Add Incoming Webhooks feature
   - Create webhook for your desired channel

## Deployment Options

### Option 1: AWS Console

1. Open AWS CloudFormation console
2. Click "Create stack" > "With new resources"
3. Upload the `iam-alerts-cloudformation.yaml` template
4. Fill in the parameters:
   - **NotificationEmail**: Your email address (leave empty if using Slack only)
   - **SlackWebhookUrl**: Your Slack webhook URL (leave empty if using email only)
   - **ExcludedUserNames**: Comma-separated list of usernames to exclude (default: portalsvc1,klam-sts-user)
   - **AlertRuleName**: Custom name for the alert rule (default: iam-user-creation-alert)
5. Review and create the stack
6. If using email, check your inbox and confirm the SNS subscription

### Option 2: AWS CLI

```bash
# Deploy with email notifications
aws cloudformation create-stack \
  --stack-name iam-activity-alerts \
  --template-body file://iam-alerts-cloudformation.yaml \
  --parameters \
    ParameterKey=NotificationEmail,ParameterValue=your-email@example.com \
    ParameterKey=ExcludedUserNames,ParameterValue="portalsvc1,klam-sts-user" \
  --capabilities CAPABILITY_IAM

# Deploy with Slack notifications
aws cloudformation create-stack \
  --stack-name iam-activity-alerts \
  --template-body file://iam-alerts-cloudformation.yaml \
  --parameters \
    ParameterKey=SlackWebhookUrl,ParameterValue=https://hooks.slack.com/services/YOUR/WEBHOOK/URL \
    ParameterKey=ExcludedUserNames,ParameterValue="portalsvc1,klam-sts-user" \
  --capabilities CAPABILITY_IAM

# Deploy with both email and Slack
aws cloudformation create-stack \
  --stack-name iam-activity-alerts \
  --template-body file://iam-alerts-cloudformation.yaml \
  --parameters \
    ParameterKey=NotificationEmail,ParameterValue=your-email@example.com \
    ParameterKey=SlackWebhookUrl,ParameterValue=https://hooks.slack.com/services/YOUR/WEBHOOK/URL \
    ParameterKey=ExcludedUserNames,ParameterValue="portalsvc1,klam-sts-user" \
  --capabilities CAPABILITY_IAM
```

### Option 3: Using a Parameters File

Create a `parameters.json` file:

```json
[
  {
    "ParameterKey": "NotificationEmail",
    "ParameterValue": "security-team@example.com"
  },
  {
    "ParameterKey": "SlackWebhookUrl",
    "ParameterValue": "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
  },
  {
    "ParameterKey": "ExcludedUserNames",
    "ParameterValue": "portalsvc1,klam-sts-user,automation-user"
  },
  {
    "ParameterKey": "AlertRuleName",
    "ParameterValue": "iam-user-creation-alert"
  }
]
```

Then deploy:

```bash
aws cloudformation create-stack \
  --stack-name iam-activity-alerts \
  --template-body file://iam-alerts-cloudformation.yaml \
  --parameters file://parameters.json \
  --capabilities CAPABILITY_IAM
```

## Monitored Events

The template monitors these IAM events:
- **CreateUser**: When a new IAM user is created
- **CreateAccessKey**: When an access key is created for any user
- **CreateLoginProfile**: When console access is enabled for a user

## Alert Information

Both email and Slack notifications include:
- Event type (CreateUser, CreateAccessKey, or CreateLoginProfile)
- Target username
- Event timestamp
- AWS Account ID
- Region
- Principal ID (who performed the action)
- Principal ARN
- Source IP address

## Customization

### Adding More Excluded Users

Update the `ExcludedUserNames` parameter with a comma-separated list:

```bash
aws cloudformation update-stack \
  --stack-name iam-activity-alerts \
  --use-previous-template \
  --parameters \
    ParameterKey=ExcludedUserNames,ParameterValue="user1,user2,user3"
```

### Monitoring Additional IAM Events

Edit the template's EventPattern section to add more events:

```yaml
eventName:
  - CreateUser
  - CreateAccessKey
  - CreateLoginProfile
  - DeleteUser  # Add this
  - DeleteAccessKey  # Add this
```

### Changing Notification Target

Update the stack parameters to change email or Slack webhook:

```bash
aws cloudformation update-stack \
  --stack-name iam-activity-alerts \
  --use-previous-template \
  --parameters \
    ParameterKey=NotificationEmail,ParameterValue=new-email@example.com \
    ParameterKey=SlackWebhookUrl,UsePreviousValue=true
```

## Testing

To test the alert:

1. Create a test IAM user (not in the excluded list):
   ```bash
   aws iam create-user --user-name test-alert-user
   ```

2. Check your email/Slack for the alert

3. Clean up:
   ```bash
   aws iam delete-user --user-name test-alert-user
   ```

## Sharing with Other Accounts

To share this template with other teams or AWS accounts:

1. **Upload to S3** (for easy access):
   ```bash
   aws s3 cp iam-alerts-cloudformation.yaml s3://your-templates-bucket/
   aws s3api put-object-acl --bucket your-templates-bucket \
     --key iam-alerts-cloudformation.yaml --acl public-read
   ```

2. **Share the S3 URL** or the template file directly

3. **Provide deployment instructions** (this README)

## Cost Considerations

- **EventBridge**: No charge for rules; charged per event matched (minimal cost)
- **Lambda** (Slack only): Charged per invocation and execution time (minimal cost, free tier eligible)
- **SNS** (email only): First 1,000 email notifications per month are free
- **CloudWatch Logs** (Slack only): Lambda logs incur minimal charges

Expected cost: **< $1/month** for typical usage

## Troubleshooting

### No alerts received

1. **Verify CloudTrail is enabled**: Management events must be logged
2. **Check SNS subscription** (email): Must confirm subscription via email
3. **Test Slack webhook**: Use `curl` to verify webhook works
4. **Check EventBridge rule**: Verify it's in "Enabled" state
5. **Review CloudWatch Logs** (Slack): Check Lambda function logs for errors

### Lambda fails (Slack)

Check CloudWatch Logs for the Lambda function:
```bash
aws logs tail /aws/lambda/iam-user-creation-alert-slack-notifier --follow
```

### Excluded users still triggering alerts

Verify the `ExcludedUserNames` parameter matches exactly (case-sensitive)

## Cleanup

To remove all resources:

```bash
aws cloudformation delete-stack --stack-name iam-activity-alerts
```

## Security Considerations

- The Slack webhook URL is marked as `NoEcho` to prevent exposure in console/CLI
- Lambda function has minimal IAM permissions (only CloudWatch Logs)
- SNS topic policy restricts publishing to EventBridge service only
- No sensitive IAM credentials are stored or transmitted

## Support

For issues or questions:
- Check AWS CloudFormation stack events for deployment errors
- Review CloudWatch Logs for Lambda execution errors
- Verify CloudTrail is properly configured and logging events
