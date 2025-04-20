# 🛡️ Cryptominer Auto Defense System for AWS GuardDuty

This project implements an automated response system to defend against cryptocurrency mining attacks detected by [Amazon GuardDuty](https://aws.amazon.com/guardduty/). When a cryptominer-related finding is triggered, the system automatically isolates the affected EC2 instance(s), shuts them down, freezes the attacker identity, and stores detailed incident logs for forensic investigation.

---

## 🔧 What It Does

Once deployed, the system performs the following actions automatically upon receiving a GuardDuty finding:

1. **Identify Compromised EC2 Instance**
2. **Suspend Auto Scaling Group (if involved)**:
   - Sets `MinSize`, `MaxSize`, and `DesiredCapacity` to `0`
3. **Network Isolation**:
   - Replaces existing security groups with a dedicated isolation group
4. **Stop the Compromised Instance**
5. **Trace Attacker Using CloudTrail**:
   - If IAM User → Disable login profile and access keys
   - If IAM Role → Remove trust policy to block further `AssumeRole`
   - If Root → Flag as **requires manual intervention**
6. **Generate and Upload JSON Incident Report**

---

## 🧠 Lambda Function Logic

The Lambda function (written in Python 3.9) executes the following internal logic:

```python
# Simplified overview
event = GuardDuty finding
instance_id = event['detail']['resource']['instanceDetails']['instanceId']

# Check if instance is in ASG
asg.describe_auto_scaling_instances()
→ If yes → asg.update_auto_scaling_group() to Min/Max/Desired = 0

# Modify all network interfaces
ec2.modify_network_interface_attribute() → Attach isolation SG

# Stop the instance
ec2.stop_instances()

# Use CloudTrail to trace attacker
→ If IAM user → freeze_user(username)
→ If AssumedRole → freeze_role(role_name)
→ If Root → mark as manual intervention

# Create incident log and upload to S3
s3.put_object(Bucket, Key, Body=JSON)
```

The function is triggered by EventBridge and runs with timeout of **300 seconds**.

---

## 📤 S3 Output Example

Each GuardDuty response generates a JSON log file and uploads it to the configured S3 bucket, under the `incidents/` folder.

**Sample file name:**
```
incidents/i-0a1b2c3d4e5f6g7h-1713590835.json
```

**Sample content:**
```json
{
  "timestamp": "2025-04-20T10:37:15.123Z",
  "guarddutyFindingType": "CryptoCurrency:EC2/BitcoinTool",
  "instanceId": "i-0a1b2c3d4e5f6g7h",
  "region": "us-east-1",
  "actor": {
    "type": "IAMUser",
    "arn": "arn:aws:iam::123456789012:user/attacker",
    "sourceIp": "198.51.100.42",
    "userAgent": "aws-cli/2.15.0"
  },
  "response": {
    "asgDisabled": true,
    "networkIsolated": true,
    "instanceStopped": true,
    "actorFrozen": true,
    "rootInvolved": false,
    "manualInterventionRequired": false
  }
}
```

This report can be used for:
- **Forensic auditing**
- **Compliance**
- **Automated analytics (e.g. Athena, Glue)**

---

## 📦 CloudFormation Resources

| Resource                     | Description                                       |
|-----------------------------|---------------------------------------------------|
| **EC2 Security Group**       | Isolation group with no inbound/outbound rules    |
| **S3 Bucket**                | Stores incident reports (versioned, unencrypted)  |
| **IAM Role**                 | Least-privileged role for Lambda execution        |
| **Lambda Function**          | Executes response logic in Python                 |
| **EventBridge Rule**         | Triggers Lambda on GuardDuty BitcoinTool findings |
| **Lambda Invoke Permission** | Grants EventBridge rights to invoke the function  |

---

## 📁 Parameters

| Parameter                  | Default                          | Description                            |
|---------------------------|----------------------------------|----------------------------------------|
| `IsolationSecurityGroupName` | `CryptominerIsolationGroup`       | Name of the isolation group            |
| `IncidentLogBucketName`      | `cryptominer-incident-logs-20250420` | Bucket name for storing JSON logs      |
| `VPCId`                      | *(Required)*                    | VPC where the isolation group is created |

---

## ✅ Deployment Steps

1. Open the AWS CloudFormation Console
2. Upload the `cryptominer_defense_template.yml` file
3. Provide the required parameters (especially `VPCId`)
4. Launch the stack

> ℹ️ Make sure that GuardDuty is **enabled** in your region and account.

---

## 🧠 Notes

- The system currently only responds to this GuardDuty finding type:  
  `CryptoCurrency:EC2/BitcoinTool`
- S3 bucket versioning is **enabled**, but encryption is **not configured** by default.
- This is a **demonstration project**, not production-hardened.
- Root-based attacks are flagged for **manual investigation**.
- You can extend the logic with SNS/Slack notifications or IAM rollback.

---

## 📜 License

MIT No Attribution License  
© 2025 AWS + Contributors
