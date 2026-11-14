# Neuralsec CloudFormation integrations

CloudFormation templates for installing Neuralsec into an AWS account or an AWS Organization. The scanner and registry templates create read-only IAM roles that only the Neuralsec broker can assume, and only with the ExternalId of your connection; the private link template creates the network resources that publish one of your private services. Nothing in them creates compute or storage.

## Templates

| Template | What it creates |
| :--- | :--- |
| `templates/cloud-scanner-role.yaml` | the read-only role `neuralsec-cloud-scanner` for one account, plus the managed policy that grants the named read actions |
| `templates/cloud-scanner-org-reader.yaml` | the role `neuralsec-cloud-scanner-org-reader` for the management or delegated administrator account, with the `organizations` read actions only |
| `templates/cloud-scanner-stackset.yaml` | a service-managed StackSet that deploys `cloud-scanner-role.yaml` to every account of the target organizational units, with auto-deployment on |
| `templates/codeartifact-read-role.yaml` | a role the broker assumes to read the CodeArtifact domains of your Neuralsec `aws_codeartifact` registries |
| `templates/private-link-service.yaml` | an internal Network Load Balancer, target group, listener and endpoint service that publish a private HTTPS service to Neuralsec over AWS PrivateLink |

### cloud-scanner-role.yaml

Parameters:

| Parameter | Description |
| :--- | :--- |
| `ExternalId` | ExternalId of the Neuralsec connection, `neuralsec-` followed by 32 lowercase hex characters |
| `NeuralsecBrokerArn` | ARN of the Neuralsec broker role, the only principal allowed to assume the role |

Outputs: `RoleArn` (ARN of the scanner role) and `RoleName` (the fixed name `neuralsec-cloud-scanner`).

The policy action list is generated from `terraform/modules/policy/actions.json` and is the same list the Terraform module uses. It has no write action and no content read.

IaC state locations are configured through the Terraform module `modules/cloud-scanner-role` only (`state_locations`). A CloudFormation template cannot build the per-location policy document that input produces, so this template grants no object read and the scanner reads no state file for a stack installed this way.

### cloud-scanner-org-reader.yaml

Parameters: `ExternalId` and `NeuralsecBrokerArn`, as above. Outputs: `RoleArn` and `RoleName` (the fixed name `neuralsec-cloud-scanner-org-reader`). Deploy it once, in the management account or the delegated administrator account, so Neuralsec can discover the accounts of the organization.

### cloud-scanner-stackset.yaml

Parameters:

| Parameter | Description |
| :--- | :--- |
| `ExternalId` | ExternalId of the Neuralsec connection |
| `NeuralsecBrokerArn` | ARN of the Neuralsec broker role |
| `TargetOrganizationalUnitIds` | comma-separated organizational unit ids that receive the role, for example `ou-abcd-11111111` |
| `TemplateURL` | required, S3 HTTPS URL of `cloud-scanner-role.yaml` for the release you install, for example `https://neuralsec-release-cloudformation-templates.s3.eu-west-1.amazonaws.com/v0.2.0/cloud-scanner-role.yaml` |

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

### private-link-service.yaml

Publishes a private HTTPS service, such as a package registry, to Neuralsec. It
creates an internal Network Load Balancer in your VPC, a TCP target group, a
listener on 443 and an endpoint service whose only allowed principal is the
Neuralsec role. TLS stays end to end: the load balancer does not terminate it.
An internal Application Load Balancer (`TargetAlbArn`) is the recommended
target: it keeps a stable ARN while its addresses change. A stack registers one
target; a service behind several addresses uses the Terraform module
`modules/private-link-service`, which takes a list.

Parameters:

| Parameter | Description |
| :--- | :--- |
| `Name` | name of the link, also the `Name` tag; 1 to 19 letters, digits or hyphens, not ending in a hyphen |
| `NeuralsecOrgId` | Neuralsec organization id, tag `neuralsec:org_id` |
| `NeuralsecOrgSlug` | Neuralsec organization slug, tag `neuralsec:org_slug` |
| `CreatedAt` | RFC 3339 timestamp, tag `neuralsec:created_at` |
| `NeuralsecPrincipalArn` | ARN of the Neuralsec role, the only allowed principal |
| `NeuralsecRegion` | region Neuralsec runs in, default `eu-west-1`, left unset; listed next to the stack region on the endpoint service when it differs |
| `VpcId` | VPC that holds the service |
| `SubnetIds` | at least two private subnets for the load balancer |
| `TargetAlbArn` | ARN of an internal Application Load Balancer, the recommended target; set exactly one of `TargetAlbArn`, `TargetIp` and `TargetInstanceId`, or `ExistingNlbArn` instead. An ALB target next to `ExistingNlbArn` is rejected |
| `TargetIp` | private IPv4 address of the service; set exactly one of `TargetAlbArn`, `TargetIp` and `TargetInstanceId` unless `ExistingNlbArn` is set, which ignores it |
| `TargetInstanceId` | instance id of the service; set exactly one of the three unless `ExistingNlbArn` is set, which ignores it |
| `TargetPort` | TCP port of the target, default 443; the listener stays on 443. With `TargetAlbArn` it must be the port of an ALB listener |
| `AcceptanceRequired` | `true` or `false`, default `true`: accept the connection request yourself in the VPC console, after checking that its endpoint ID matches the one shown in the Neuralsec app |
| `ExistingNlbArn` | ARN of an existing internal Network Load Balancer; when set, the stack creates only the endpoint service and ignores `TargetIp` and `TargetInstanceId` |

Outputs: `ServiceName` (paste it into Neuralsec), `EndpointServiceId`, `NlbArn`
and `NlbSecurityGroupId`. Allow `NlbSecurityGroupId` on the target port in the
security group of the service itself, and inbound on the target port in the
security group of an Application Load Balancer target; that is the one manual
step.

Every taggable resource carries `neuralsec:component`,
`neuralsec:integration-version`, `neuralsec:org_id`, `neuralsec:org_slug`,
`neuralsec:created_at` and `Name`.

## Deploying

Neuralsec shows the ExternalId and the broker ARN when you create the connection. Every release is also published, read-only and over HTTPS only, at `https://neuralsec-release-cloudformation-templates.s3.eu-west-1.amazonaws.com/<tag>/<template>` (for example `https://neuralsec-release-cloudformation-templates.s3.eu-west-1.amazonaws.com/v0.2.0/cloud-scanner-role.yaml`), and the rolling `latest/` prefix holds the newest release. Pin a tag for production installs.

### Console quick-create

Use the Quick-create link Neuralsec shows for your connection: it opens the CloudFormation console with the published template URL and both parameters filled in. To do it by hand, choose Create stack with new resources, give the published template URL (or upload the template file), fill in `ExternalId` and `NeuralsecBrokerArn`, tick the acknowledgement that the stack creates named IAM resources, and create the stack.

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
3. Deploy `cloud-scanner-stackset.yaml` as a normal stack in the same account, with `TemplateURL` set to the published `cloud-scanner-role.yaml` of the same release (or to your own copy in S3) and `TargetOrganizationalUnitIds` set to the OUs to scan:

```
aws cloudformation deploy \
  --stack-name neuralsec-cloud-scanner-stackset \
  --template-file templates/cloud-scanner-stackset.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    ExternalId=neuralsec-0123456789abcdef0123456789abcdef \
    NeuralsecBrokerArn=arn:aws:iam::111111111111:role/neuralsec-broker \
    TargetOrganizationalUnitIds=ou-abcd-11111111 \
    TemplateURL=https://neuralsec-release-cloudformation-templates.s3.eu-west-1.amazonaws.com/v0.2.0/cloud-scanner-role.yaml
```

Service-managed StackSets do not deploy to the management account itself. Deploy `cloud-scanner-role.yaml` there as a normal stack if you want it scanned.

## EKS

These templates do not create EKS access entries: CloudFormation cannot expand a list of cluster names into one `AWS::EKS::AccessEntry` per cluster. Accounts with EKS clusters use the Terraform module `modules/cloud-scanner-role` from `neuralsec-io/terraform-aws-neuralsec-integrations`, which creates the access entries.

The scanner connects to the Kubernetes API from two address families. It prefers IPv6 towards a dual-stack endpoint, and falls back to IPv4.

| Family | Address | Allowed on |
| :--- | :--- | :--- |
| IPv6 | `2a05:d018:e21:f00::/56` | a dual-stack (`ipv6`) cluster created after October 2024 |
| IPv4 | `3.248.10.74/32`, `34.241.8.107/32`, `52.16.218.0/32` | every cluster |

The Terraform module exposes them as `scanner_egress_ipv6_cidrs`, `scanner_egress_ipv4_cidrs` and `scanner_egress_cidrs` (both families, IPv6 first). If the cluster's public endpoint is CIDR restricted, allow the list for your cluster instead of opening the endpoint to `0.0.0.0/0`.

Check the family of the cluster:

```
aws eks describe-cluster --name example-cluster --region eu-west-1 \
  --query 'cluster.kubernetesNetworkConfig.ipFamily'
```

An `ipv6` cluster created after October 2024 accepts IPv4 and IPv6 CIDRs in the same `publicAccessCidrs` list, and its default is `0.0.0.0/0` plus `::/0`. Allow both scanner families there, the `/56` and the three `/32`s, because the scanner prefers IPv6 and falls back to IPv4. Every other cluster (IPv4, or an `ipv6` cluster created before October 2024) accepts IPv4 CIDRs only, so allow the three `/32`s.

For a dual-stack IPv6 cluster created after October 2024:

```
aws eks update-cluster-config \
  --name example-cluster \
  --region eu-west-1 \
  --resources-vpc-config endpointPublicAccess=true,publicAccessCidrs=2001:db8:1234:5600::/56,198.51.100.0/24,2a05:d018:e21:f00::/56,3.248.10.74/32,34.241.8.107/32,52.16.218.0/32
```

For every other cluster:

```
aws eks update-cluster-config \
  --name example-cluster \
  --region eu-west-1 \
  --resources-vpc-config endpointPublicAccess=true,publicAccessCidrs=198.51.100.0/24,3.248.10.74/32,34.241.8.107/32,52.16.218.0/32
```

`publicAccessCidrs` replaces the whole list, so append the scanner CIDRs to the CIDRs you already allow instead of replacing them (the placeholders `2001:db8:1234:5600::/56` and `198.51.100.0/24` above stand for yours).

A cluster with a private-only endpoint has no public endpoint, so no allow list can reach it from outside. The scanner reports such a cluster as `kubernetes_api_unreadable`; reading the Kubernetes API from inside the VPC is planned. The rest of the cluster's AWS metadata is still inventoried.

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
| `private_link_single_principal` | every endpoint service allows only the `NeuralsecPrincipalArn` parameter reference, never `*` and never a literal ARN | the private service is reachable only by the role Neuralsec names at setup time |

### Running cfn-guard locally

Install cfn-guard 3.2.1 from the [release page](https://github.com/aws-cloudformation/cloudformation-guard/releases/tag/3.2.1), then from the repository root:

```
cfn-guard validate -d templates -r rules/neuralsec.guard --show-summary all
```

In the private source repository CI runs the same command with the paths `cloudformation/templates` and `cloudformation/rules/neuralsec.guard`, next to `cfn-lint` on the templates.

## Versioning

Templates are released together with the Terraform modules under one `v*` tag, and the public repository carries every tag. Release notes are on the [releases page](https://github.com/neuralsec-io/cloudformation-neuralsec-integrations/releases). Template parameters and outputs are part of the release contract: they do not change within a release line.
