# Neuralsec CloudFormation integrations

CloudFormation templates for installing Neuralsec into an AWS account or an AWS Organization. Every template creates read-only IAM roles that only the Neuralsec broker can assume, and only with the ExternalId of your connection. Nothing in them creates compute, storage or network resources.

## Templates

| Template | What it creates |
| :--- | :--- |
| `templates/cloud-scanner-role.yaml` | the read-only role `neuralsec-cloud-scanner` for one account, plus the managed policy that grants the named read actions |
| `templates/cloud-scanner-org-reader.yaml` | the role `neuralsec-cloud-scanner-org-reader` for the management or delegated administrator account, with the `organizations` read actions only |
| `templates/cloud-scanner-stackset.yaml` | a service-managed StackSet that deploys `cloud-scanner-role.yaml` to every account of the target organizational units, with auto-deployment on |
| `templates/codeartifact-read-role.yaml` | a role the broker assumes to read the CodeArtifact domains of your Neuralsec `aws_codeartifact` registries |

### cloud-scanner-role.yaml

Parameters:

| Parameter | Description |
| :--- | :--- |
| `ExternalId` | ExternalId of the Neuralsec connection, `neuralsec-` followed by 32 lowercase hex characters |
| `NeuralsecBrokerArn` | ARN of the Neuralsec broker role, the only principal allowed to assume the role |

Outputs: `RoleArn` (ARN of the scanner role) and `RoleName` (the fixed name `neuralsec-cloud-scanner`).

The policy action list is generated from `terraform/modules/policy/actions.json` and is the same list the Terraform module uses. It has no write action and no content read.

### cloud-scanner-org-reader.yaml

Parameters: `ExternalId` and `NeuralsecBrokerArn`, as above. Outputs: `RoleArn` and `RoleName` (the fixed name `neuralsec-cloud-scanner-org-reader`). Deploy it once, in the management account or the delegated administrator account, so Neuralsec can discover the accounts of the organization.

### cloud-scanner-stackset.yaml

Parameters:

| Parameter | Description |
| :--- | :--- |
| `ExternalId` | ExternalId of the Neuralsec connection |
| `NeuralsecBrokerArn` | ARN of the Neuralsec broker role |
| `TargetOrganizationalUnitIds` | comma-separated organizational unit ids that receive the role, for example `ou-abcd-11111111` |
| `TemplateURL` | required, S3 HTTPS URL of `cloud-scanner-role.yaml` for the release you install, in a bucket the deploying account can read |

It creates the StackSet `neuralsec-cloud-scanner-role` (service-managed, `CAPABILITY_NAMED_IAM`, deployed to the region of the stack). Accounts added to the target OUs later receive the role automatically; stacks are removed when an account leaves the OUs (`RetainStacksOnAccountRemoval: false`). It has no outputs.

### codeartifact-read-role.yaml

Parameters:

| Parameter | Description |
| :--- | :--- |
| `ExternalIds` | ExternalId of every Neuralsec CodeArtifact registry that shares the role |
| `NeuralsecBrokerArn` | ARN of the Neuralsec broker role |
| `RoleName` | role name, default `neuralsec-registry-read` |
| `RolePath` | IAM path, default `/` |
| `DomainArns` | CodeArtifact domain ARNs |
| `RepositoryArns` | repository ARNs of the same domains, each ending in `/*` |

Output: `RoleArn`. The inline policy `neuralsec-codeartifact-read` grants `codeartifact:GetAuthorizationToken` on the domains, `codeartifact:GetRepositoryEndpoint` and `codeartifact:ReadFromRepository` on the repositories, and `sts:GetServiceBearerToken` restricted to `codeartifact.amazonaws.com`. When another registry starts using the same role, update the stack with its ExternalId instead of creating a second stack.

## Deploying

Neuralsec shows the ExternalId and the broker ARN when you create the connection. Take the templates from a tagged release of this repository (each release carries every template), and pin that tag for production installs.

### Console quick-create

Open the CloudFormation console, choose Create stack with new resources, upload the template file, fill in `ExternalId` and `NeuralsecBrokerArn`, tick the acknowledgement that the stack creates named IAM resources, and create the stack.

### AWS CLI

```
aws cloudformation deploy \
  --stack-name neuralsec-cloud-scanner \
  --template-file templates/cloud-scanner-role.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    ExternalId=neuralsec-0123456789abcdef0123456789abcdef \
    NeuralsecBrokerArn=arn:aws:iam::111111111111:role/neuralsec-broker
```

The values above are placeholders. Use the ones Neuralsec shows for your connection.

### AWS Organization with StackSets

1. In the management account (or a delegated administrator), enable trusted access for CloudFormation StackSets in AWS Organizations.
2. Deploy `cloud-scanner-org-reader.yaml` as a normal stack in the management or delegated administrator account.
3. Upload `cloud-scanner-role.yaml` from the release to an S3 bucket in that account (StackSets read their template from S3), for example `aws s3 cp templates/cloud-scanner-role.yaml s3://<your-bucket>/neuralsec/v0.1.0/cloud-scanner-role.yaml`.
4. Deploy `cloud-scanner-stackset.yaml` as a normal stack in the same account, with `TemplateURL` set to the S3 HTTPS URL of that object and `TargetOrganizationalUnitIds` set to the OUs to scan:

```
aws cloudformation deploy \
  --stack-name neuralsec-cloud-scanner-stackset \
  --template-file templates/cloud-scanner-stackset.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    ExternalId=neuralsec-0123456789abcdef0123456789abcdef \
    NeuralsecBrokerArn=arn:aws:iam::111111111111:role/neuralsec-broker \
    TargetOrganizationalUnitIds=ou-abcd-11111111 \
    TemplateURL=https://<your-bucket>.s3.<region>.amazonaws.com/neuralsec/v0.1.0/cloud-scanner-role.yaml
```

Service-managed StackSets do not deploy to the management account itself. Deploy `cloud-scanner-role.yaml` there as a normal stack if you want it scanned.

## EKS

These templates do not create EKS access entries: CloudFormation cannot expand a list of cluster names into one `AWS::EKS::AccessEntry` per cluster. Accounts with EKS clusters use the Terraform module `modules/cloud-scanner-role` from `neuralsec-io/terraform-aws-neuralsec-integrations`, which creates the access entries.

## Guard rules

`rules/neuralsec.guard` holds [cfn-guard](https://github.com/aws-cloudformation/cloudformation-guard) rules that every template must pass before a release. They turn the "read-only, no customer content" promise into a check you can read and run yourself.

| Rule | What it enforces | Why |
| :--- | :--- | :--- |
| `no_wildcard_actions` | no managed policy action contains a `*` | a wildcard hides permissions a reviewer cannot see; the scanner holds named actions only |
| `only_named_read_actions` | every managed policy action is `List`, `Describe`, `Get`, `BatchGet`, `Search`, `Select` or `Lookup` on a service, plus exactly two exceptions: `iam:GenerateCredentialReport` (starts the credential report) and `apigateway:GET` (API Gateway names its read action after the HTTP verb) | keeps the role read-only; a new write or mutate action fails the check |
| `no_content_read_actions` | no action that returns customer content, in any letter case: `s3:GetObject`, `s3:GetObjectVersion`, `s3:SelectObjectContent`, `secretsmanager:GetSecretValue`, `secretsmanager:BatchGetSecretValue`, `ssm:GetParameter*`, `kms:Decrypt`, `logs:GetLogEvents`, `logs:FilterLogEvents`, `dynamodb:GetItem`, `dynamodb:Scan`, `dynamodb:Query`, `sqs:ReceiveMessage`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer` | some read verbs return object bodies, secret values, log lines, items or messages; Neuralsec reads metadata only |
| `no_wildcard_role_trust_actions` | no role trust policy action contains a `*` | the trust policy may only allow `sts:AssumeRole` |
| `role_trust_only_neuralsec_broker` | every role trust statement is an Allow of `sts:AssumeRole` whose principal is the `NeuralsecBrokerArn` parameter reference; a literal ARN, `*` or any other principal fails | only the Neuralsec broker can ever assume the role |
| `role_trust_requires_external_id` | every statement of a role trust policy has an `sts:ExternalId` condition | the broker must present the ExternalId of your connection, so no other Neuralsec customer can use the role (confused deputy protection), and there is no condition-free fallback |

### Running cfn-guard locally

Install cfn-guard 3.2.1 from the [release page](https://github.com/aws-cloudformation/cloudformation-guard/releases/tag/3.2.1), then from the repository root:

```
cfn-guard validate -d templates -r rules/neuralsec.guard --show-summary all
```

In the private source repository CI runs the same command with the paths `cloudformation/templates` and `cloudformation/rules/neuralsec.guard`, next to `cfn-lint` on the templates.

## Versioning

Templates are released together with the Terraform modules under one `v*` tag, and the public repository carries every tag. Release notes are on the [releases page](https://github.com/neuralsec-io/cloudformation-neuralsec-integrations/releases). Template parameters and outputs are part of the release contract: they do not change within a release line.
