# IAM Investigation and Isolation
All actions of users that are suspicious can be investigated with [Amazon Detective](https://aws.amazon.com/detective/). It can help you do:
- Threat Hunting
- Triage Security findings
- Investigate incidents 

For an overview - refer to this [introduction](https://www.youtube.com/watch?v=Vt8xMRnt1OA). 

Detective allows you to quickly analyze CloudTrail and VPC Flow logs to review role session information. It can identify: 
- Which resource assumed a role
- How the resource was used for
- What activities were performed
- Whether role chaining was used 

In order to effectively use this - it should be enabled in your security account and member accounts should be onboarded to the central account in your organization. Once you have identified that a role was used maliciously you need to: 

- [Suspend or disable the IAM user](https://docs.aws.amazon.com/singlesignon/latest/userguide/disableuser.html)
- [Revoke any sessions for a specific IAM role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_revoke-sessions.html)

Note that that [ temporary security credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_control-access_disable-perms.html) are valid until they expire, and they cannot be revoked. However, because permissions are evaluated each time an AWS request is made using the credentials, you can achieve the effect of revoking the credentials by changing the permissions for the credentials even after they have been issued. If you remove all permissions from the temporary security credentials, subsequent AWS requests that use those credentials will fail. The mechanisms for changing or removing the permissions assigned to temporary security credentials are explained in this page.

You cannot change the permissions for an AWS account root user. Likewise, you cannot change the permissions for the temporary security credentials that were created by calling GetFederationToken or GetSessionToken while signed in as the root user. For this reason, we recommend that you do not call GetFederationToken or GetSessionToken as a root user. 


## Denying access for a user through CMD 
Note that to disable access for a particular user (and not a role!). You can attach the "AWSDenyAll" managed policy. By running the following command to attach the policy: `aws iam attach-user-policy --user-name bobby --policy-arn arn:aws:iam::aws:policy/AWSDenyAll`

```json
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Action": [
                        "*"
                    ],
                    "Effect": "Deny",
                    "Resource": "*"
                }
            ]
        },
        "VersionId": "v1",
        "IsDefaultVersion": true,
        "CreateDate": "2019-05-01T22:36:14Z"
    }
}

```

## Alerting on attaching AWSDenyAll
The following EventBus event matches attaching the AWSDenyAll. You can then forward this to the chatbot SNS topic (note that this is the same one we use for SecurityHub as mentioned in [securityhub-to-chatbot](../securityhub-to-chatbot)) 

```json
{
  "source": ["aws.iam"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["iam.amazonaws.com"],
    "eventName": ["AttachUserPolicy"],
    "requestParameters": {
      "policyArn": ["arn:aws:iam::aws:policy/AWSDenyPolicy"]
    }
  }
}
```

Make sure that the SNS topic which is used by the Chatbot has permission to post to SNS:

```json
{

    
      "Sid": "556f5d32-e909-44e3-bf9a-170e8c7d3fe3",
      "Effect": "Allow",
      "Principal": {
        "Service": "events.amazonaws.com"
      },
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:*:711137749324:security-events"
    
  
}

```