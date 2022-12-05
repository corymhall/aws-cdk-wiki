# CDK Security and Safety Dev Guide


## Introduction

The CDK can be used to automatically make changes to any configuration and service in your
AWS account. This makes it a powerful tool, and it can make developers extremely efficient.
The more things can be configured via CDK, the more efficient developers can get. At the
same time, organizations commonly have a desire to limit the services that developers have
access to, with the goal to guarantee that they cannot tamper with compliance or cost
control measures set up in the account. There is commonly a tension between security and
productivity, and in a large organization there are typically different actors that try to
safeguard both sides of this equation.

This page will take you through choices that you can make with respect to CDK security and
safety, and will give some of our recommendations that we think will satisfy most use
cases without impacting developer productivity. It will cover the following topics:

* What permissions to extend to CDK deployments
* How to control the permissions of CDK deployments via IAM identities and policies
* How to use CDK to configure the IAM identities and policies of deployed applications
* Using Permissions Boundaries with CDK

## What permissions should I give to CDK deployments?

There are generally two strategies for coming up with a set of developer permissions:

* Figuring out what IAM permissions are required a priori and only allowing those (allow listing)
* Putting guard rails and compliance checks in place and allowing everything else that does not
  interfere with the guard rails (deny listing)

Finally, there’s a choice to make on whether or not to allow CDK deployments to create new Roles or not.

This section is written with the understanding that limiting permissions by itself is almost never
a goal, but a means to an end: preventing accidental or purposeful interruption of service, data
loss, data leaks, or large bills. 

### Allow listing

In general, it’s preferred to extend the smallest set of permissions that suffices for the task at
hand, and no more. CDK is no different, except that it comes with two complications:

* Coming up with an exhaustive list of required permissions is hard
* If you want to be very fine grained, the permissions may become too long to fit into the maximum
  length of IAM policy documents
* Getting the list wrong may severely impact developer productivity

Ultimately, CloudFormation just executes a set of AWS API calls in order using the permissions that
are given to it. What permissions are necessary at any point in time depends on many factors: 

* The services that developers plan to use, and specifically what properties of each of the resources
  they plan to use (these change over time)
* The current state of the stack (these may be different in different stages)
* Whether the deployment runs into problems and needs to roll back (requires Delete permissions in addition to Create).

If the list of permissions is incomplete, human intervention is inevitable:

* If you discover the missing permissions during roll forward, nothing much happens but the developer
  team and the security team now need to schedule time to discuss the new permissions.
* In the worst case, your deployment rolls back and the permissions to apply the roll back are missing.
  Depending exactly on what permissions these are and what the state is, this may leave your
  CloudFormation stack in a state that will require a lot of manual work to recover from
  (and the negotiation process from the previous paragraph needs to happen as well).

As you can see, allow listing permissions for infrastructure deployments is a tricky process and we
do not recommend using this strategy.

### Deny listing

An alternative approach taken by many companies is to put guard rails, compliance rules, auditing, and
monitoring in place using systems such as AWS Config, AWS CloudTrail, AWS Security Hub, AWS CloudFormation
Hooks, and others, and allow developers to do everything, except tampering with the existing validation
mechanisms.

This is another way to have guarantees that their compliance goals are met, while leaving the developers
with the freedom to make choices and be fast, as long as they stay within policy.

### Allowing creation of IAM Roles without privilege escalation

A question that comes up a lot is whether developers should be allowed to create new IAM Roles.
As part of normal operation, the CDK will automatically create roles and provision them with
least-privilege permissions. This is both desirable from a security standard, as well as efficient:
this is default CDK behavior and it takes effort to turn it off and replace the automatically
generated roles with human-defined roles.

Taking away the CDK’s ability to create these roles for you both slows down application development
and worsens your security posture:

* The Roles will need to be created by hand, slowing down the development cycle.
* Because Roles need to be created by hand, multiple logical Roles tend to be combined into one
  physical Role to keep the process manageable for humans. This runs counter to the least-privilege principle.
* Because these Roles are created before the deployment flow, the resources they need to refer to
  will not exist yet, commonly forcing the human policy authors to use wildcards. This runs counter
  to the least-privilege principle.
* A security department may mandate that all resources need to be given a predictable name to avoid
  the policy wildcards in hand-created roles, but this interferes with CloudFormation’s ability to
  replace resources when necessary and may slow down or block development when the need for this arises.
  It is a CDK best practice to allow CloudFormation to pick unique resource names for you.
* It makes it impossible to perform [continuous delivery](https://aws.amazon.com/builders-library/going-faster-with-continuous-delivery)
  (CD) since a manual action must be performed prior to every deployment.

Commonly, the rule that application developers are not allowed to create Roles is a mechanism to
guarantee that developers cannot elevate their own privileges: allowing them to create Roles would
allow them to create Roles with policies that escape the limitations set in place.

An alternative mechanism that provides the same guarantees without taking away developer’s abilities
to create Roles, is using either *Permissions Boundaries* or *Service Control Policies* (SCPs).
By attaching a Permissions Boundary to the developer’s Roles you are limiting both the actions the
developer can perform directly, as well as the actions that can be performed by any Roles they create.
SCPs allow you to restrict actions for *every* user and role in an AWS account. Using one of these
mechanisms, you can allow developers to create Roles without the possibility of them gaining permissions
that are not allowed by the Boundary. We recommend using Permissions Boundaries or SCPs whenever
possible. See the section *Using Permissions Boundaries with CDK* for more information on how to
configure these.

## Controlling the permissions used by CDK deployments

CDK uses CloudFormation to deploy changes. Every deployment involves an *Actor* (either a developer,
or an automated system) that starts a *CloudFormation deployment.* In the course of doing this, the
actor will assume one or more *IAM Identities* (User or Roles) and optionally pass a Role to
CloudFormation. What Roles and asset containers are used by the CDK for a Stack deployment is
determined by the [Stack Synthesizer](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib-readme.html#stack-synthesizers)
defined for the Stack. 

### DefaultStackSynthesizer

The  [`DefaultStackSynthesizer`](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.DefaultStackSynthesizer.html)
is the default stack synthesizer for CDK v2 and is the synthesizer that most people will end up using.
When using the DefaultStackSynthesizer you will first need to *[bootstrap](#bootstrapping)*
your AWS environment. *Bootstrapping* is the one-time process of creating the IAM Roles and Asset
containers (S3 Bucket and ECR Repository) that the Synthesizer expects to exist. 

#### Deployment Flow

This guide will first walk through the DefaultStackSynthesizer deployment flow, detailing the IAM
Roles and permissions that are used. You can jump to the next section on *[bootstrapping](#bootstrapping)*
for the details on how these mentioned resources are created. 

##### Authentication

The first step is to authenticate to AWS and obtain the first IAM identity in the chain:

* **IAM User credentials**: these are the first set of IAM User credentials used to authenticate to
  AWS. These credentials are a long-lived AWS Access Key, usually configured in the credentials file
  in your home directory (`$HOME/.aws/credentials`). 
    * Optionally, you can configure the AWS CLIs and SDKs to immediately **assume a Role** after
      authenticating as a User (see [Using an IAM Role in the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-role.html)
      for more information).
* Alternatively, you may be using a **Single Sign-On** provider (like [AWS IAM Identity Center](https://aws.amazon.com/iam/identity-center/)).
  In this style you authenticate using some provider-specific mechanism, and the provider supplies you
  with short-lived session credentials that authorize you to act as a pre-defined IAM Role.
* Other services inside AWS, like a [CodePipeline](https://docs.aws.amazon.com/cdk/v2/guide/cdk_pipeline.html),
  will have an IAM Execution Role associated with them already.

[Image: Image.jpg]

##### **CloudFormation Deployment**

The next step after authenticating is to *assume* the **CDK Deploy Role**, and *pass* the
**CFN Execution Role** to CloudFormation. The CFN Execution Role is a
[CloudFormation service role](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-iam-servicerole.html)
that is used by CloudFormation to perform the Stack deployment. When a CloudFormation service role is
passed to CloudFormation, the permissions used for the deployment are the permissions assigned to the
service role, not the permissions of the user that initiated the deployment. Using this method allows
for a couple of benefits over direct deployments:

* Access to deployments are controlled by limiting who can assume the **CDK Deploy Role**. 
* Permissions for creating AWS resources are given to the **CFN Execution Role** instead of individual users.
* Use of a Deploy Role allows cross-account deployments, as the Role can be assumed by AWS Identities in a different account. 

##### **Assets and Lookups**

In addition to starting a deployment, the CDK performs a couple more actions that require permissions,
and for which certain roles are used. These have been left out of the above diagram in order to not
overcrowd it, but are presented here for completeness.

* **Upload S3 and ECR assets**: CDK can push code and auxiliary files and Docker images to the cloud
  to be used by a deployment. We call this mechanism *[assets](https://docs.aws.amazon.com/cdk/v2/guide/assets.html)*.
  An S3 Bucket and ECR Repository are created during bootstrapping to hold these assets, along with two
  IAM Roles (**file-publishing-role** and **image-publishing-role**) that have permissions to write
  to these resources.
    * The CDK CLI will assume the **file-publishing-role** and **image-publishing-role** Roles and use their
      permissions to publish to the asset Bucket and Repository. The current IAM identity must have
      permissions to assume these roles, and the CFN Execution Role must have sufficient permissions to
      read from the Bucket and Repository.
* **Lookups**: CDK can make AWS calls to look up the configuration of certain resources at synth time.
  This mechanism is called [environmental context lookups](https://docs.aws.amazon.com/cdk/v2/guide/context.html). 
    * The CDK CLI will assume a role called the **lookup-role**. The current **** IAM identity must
      have permissions to assume this role, and the lookup role itself must have permissions to make the
      AWS calls that the CDK app expects. By default, the role has the `ReadOnly` managed policy attached.

#### Bootstrapping

The CDK CLI comes with a CloudFormation template that gets used by default when you run `cdk bootstrap`.
You can [customize this template](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping.html#bootstrapping-customizing)
or create equivalent resources using another mechanism. 

When using the `DefaultStackSynthesizer`, the bootstrapping step is where you do access control.

* You control the permissions that CDK Deployments execute under by selecting a managed policy for
  the **CFN Execution Role**, and optionally by configuring a **Permissions Boundary**. By default,
  the Role will have `AdministratorAccess` permissions, and no Permissions Boundary.
* You control who can perform CDK Deployments by controlling who has permissions to `AssumeRole`
  the **CDK Deploy Role, File Publishing Role**, and **Image Publish Role**. The default
  AssumeRolePolicyDocuments of these roles allows all other identities in the same AWS Account with
  the appropriate `AssumeRole` policy statements to assume them.

To configure the IAM identities in your AWS account with enough permissions to assume the CDK
Bootstrapped roles, add a policy with the following policy statement to the identities:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AssumeCDKRoles",
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "*",
    "Condition": {
      "ForAnyValue:StringEquals": {
        "iam:ResourceTag/aws-cdk:bootstrap-role": [
          "image-publishing",
          "file-publishing",
          "deploy",
          "lookup"
        ]
      }
    }
  }]
}
```

If you are using any compliance control mechanisms, the bootstrapping phase is where you would set
these up. Make sure that the CFN Execution Role or developer-accessible identities have policies or
permission boundaries attached that prevent bypass of the mechanisms you put in place. What policies
are appropriate for that depends on the specific mechanisms that you pick. 

##### Policies for bootstrapping

The default bootstrap stack that comes with CDK currently contains a specific set of resources that
require certain permissions, but we may change and expand the contents of the bootstrap stack over
time. We cannot predict what features and resources might be added in the future.

Bootstrapping itself is a one-time operation performed by AWS account administrators, and we
recommend executing it using `AdministratorAccess` privileges. This makes sure you are safe against
future changes, and since the bootstrapping process will—by design—create new Roles with arbitrary
policies anyway, there is no real benefit to restricting the permissions.

If you want to limit to the services currently used by the bootstrap stack, that policy looks like this:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "cloudformation:*",
                "ecr:*",
                "ssm:*",
                "s3:*",
                "iam:*"
            ],
            "Resource": "*"
        }
    ]
}
```

### CliCredentialsStackSynthesizer

An alternative to using the DefaultStackSynthesizer and the associated deployment flow is to use the
`CliCredentialsStackSynthesizer`. When this synthesizer is used the CLI will still perform the same
actions, but it will use local credentials instead of assuming any IAM roles. This means that the
local credentials need to have a superset of the permissions that the bootstrap roles contain. When
using the `CliCredentialsStackSynthesizer` it is still possible to use a separate CloudFormation
execution role by specifying the role in the `--role-arn` CLI argument.

[Image: Image.jpg]

A couple of notes about using the `CliCredentialsStackSynthesizer` or the `LegacyStackSynthesizer`:

* The CloudFormation call is triggered directly and no service role is passed. The current identity
  needs to have access to CloudFormation, along with any permissions required to make the necessary
  updates to AWS resources. The deployment permissions will be limited to the permissions granted to
  the user performing the deployment.

* Asset publishing and CloudFormation deployments will be done using the current IAM identity; this
  identity must have sufficient permissions to both read from and write to the asset Bucket and Repository. 
* Lookups are performed using the current IAM identity, and lookups are subject to its policies.

## Working with IAM identities and policies in CDK

How you work with IAM identities in CDK depends on whether you allow CDK deployments to create Roles
and Policies.

* If you allow CDK to create Roles and Policies, the CDK will automatically create execution Roles
  for all AWS resources that require such Roles (EC2 Instances, Lambda Functions, CodeBuild Projects,
  Event Rules, etc...). CDK will also provision these roles with least-privilege policies based on
  the methods called on your code: `grant` methods, service integrations, etc. This is the
  recommended setup.
* If you do not allow CDK to create Roles, they will have to be created with appropriate policies by
  a human operator, in and out-of-band process. They are then later explicitly referenced by
  application developers later on when they are writing their CDK apps.

See the section *[Allowing creation of IAM Roles without privilege escalation](#allowing-creation-of-iam-roles-without-privilege-escalation)* 
for some discussion on making this decision.

### Letting the CDK manage IAM Roles and Policies

By default, the CDK will automatically create execution Roles for all AWS resources that require
such Roles (EC2 Instances, Lambda Functions, CodeBuild Projects, Event Rules, etc...). CDK will also
provision these roles with least-privilege policies based on the methods called on your code: `grant`
methods, service integrations, etc. 

This makes it easier for developers to follow security best practices and apply least privilege
permissions. For example, using the CDK to create an AWS Lambda Function that can read from an
encrypted AWS S3 Bucket can be done by using `bucket.grantRead(handler)`.


```ts
import {
  aws_s3 as s3,
  aws_lambda as lambda,
  aws_kms as kms,
  Stack,
} from 'aws-cdk-lib';

const stack = new Stack(app, 'LambdaStack');
const key = new kms.Key(stack, 'BucketKey');
const bucket = new s3.Bucket(stack, 'Bucket', {
  encryptionKey: key,
});
const handler = new lambda.Function(stack, 'Handler', {
  runtime: lambda.Runtime.NODEJS_16_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset(path.join(__dirname, 'lambda-handler')),
});

bucket.grantRead(handler);
```

The CDK will create a unique IAM Role for this specific Lambda Function and give least privilege
permissions to read from the S3 Bucket and decrypt using the KMS Key on the Bucket.

If desired, it is also possible to directly manipulate the policy of the Lambda Role:

```ts
import { aws_iam as iam } from 'aws-cdk-lib';

handler.addToRolePolicy(new iam.PolicyStatement({
  actions: ['s3:GetObject', 's3:List*'],
  resources: [
    bucket.bucketArn,
    bucket.arnForObjects('*'),
  ]
}));
```

But the use of `grant` methods is recommended whenever available.

### Using Roles pre-created by human operators

If CDK is not allowed to create IAM roles, the Roles need to be created by humans and passed into
the CDK application. You can do this in one of two ways:

* Reference and manage all roles by hand
* Use the CDK *Role Customization* feature to automatically generate a report of all Roles and
* policies, and substitute precreated roles for them.

#### Referencing and managing Roles by hand

Constructs that need a Role have an optional `role` property where you can pass in a Role object you
obtain from elsewhere. To reference pre-existing roles, you will call `Role.fromRoleName()`. The
example from above, using the Lambda Function using an existing Role, looks like this:

```ts
const existingRole = Role.fromRoleName(stack, 'Role', 'my-pre-existing-role', {
  // Important to pass 'mutable: false' so CDK will not attempt to add policies to it
  mutable: false,
});

const key = new kms.Key(stack, 'BucketKey');
const bucket = new s3.Bucket(stack, 'Bucket', {
  encryptionKey: key,
});
const handler = new lambda.Function(stack, 'Handler', {
  runtime: lambda.Runtime.NODEJS_16_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset(path.join(__dirname, 'lambda-handler')),
  
  // Pass the pre-existing role in
  role: existingRole,
});

// No need to call bucket.grantRead(), the Role is not mutable anyway
// bucket.grantRead(handler);
```

The Role creator and application developer have to communicate to agree on a set of IAM policies for
the Role.

#### Using the Customize Roles feature to generate a report and supply role names

CDK can generate a report of the Roles and policies that it *would* create, without actually
creating them. The application developer can hand this report to the authority in charge of creating
IAM Roles, wait for them to come back with physical Role names, and plug those into the application.

Start by putting `Role.customizeRoles()` somewhere near the top of your application. For example,
in the example from the previous section:

```ts
const stack = new Stack(app, 'LambdaStack');

// Add this to prevent Role synthesis
iam.Role.customizeRoles(stack);

// The rest of the program is the same
const key = new kms.Key(stack, 'BucketKey');
const bucket = new s3.Bucket(stack, 'Bucket', {
  encryptionKey: key,
});
const handler = new lambda.Function(stack, 'Handler', {
  runtime: lambda.Runtime.NODEJS_16_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset(path.join(__dirname, 'lambda-handler')),
});

// The grantRead() is still important. Even though it actually doesn't mutate
// any policies, it indicates the need for them.
bucket.grantRead(handler);
```

When the application is synthesized it will throw an error indicating that you need to provide the
precreated role name to `Role.customizeRoles()`. The report generated for the above example is below.

```txt
<missing role> (LambdaStack/Handler/ServiceRole)

AssumeRole Policy:
[
  {
    "Action": "sts:AssumeRole",
    "Effect": "Allow",
    "Principal": {
      "Service": "lambda.amazonaws.com"
    }
  }
]

Managed Policy ARNs:
[
  "arn:(PARTITION):iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
]

Managed Policies Statements:
NONE

Identity Policy Statements:
[
  {
    "Action": [
      "s3:GetObject*",
      "s3:GetBucket*",
      "s3:List*"
    ],
    "Effect": "Allow",
    "Resource": [
      "(LambdaStack/Bucket/Resource.Arn)",
      "(LambdaStack/Bucket/Resource.Arn)/*"
    ]
  }
]
```

Once the role has been created the name can be provided along with the construct path. For example
if the name for the role that is created for `LambdaStack/Handler/ServiceRole` is `lambda-service-role`,
you would update `Role.customizeRoles()` to contain:

```ts
const stack = new Stack(app, 'LambdaStack');

// Add this to prevent Role synthesis
iam.Role.customizeRoles(stack, {
  usePrecreatedRoles: {
    'LambdaStack/Handler/ServiceRole': 'lambda-service-role',
  },
});
```

The CDK will now use the precreated role name anywhere that the role is referenced in the CDK
application. It will also continue to generate the report so that any future policy changes can be referenced.

You will notice that the reference to the S3 Bucket ARN in the report is rendered as
`(LambdaStack/Bucket/Resource.Arn)` instead of the actual ARN of the bucket. This is because the
Bucket ARN is a deploy time value that is not known at synthesis (the bucket hasn’t been created
yet). This is another example of why we recommend allowing CDK to manage IAM Roles and
permissions. In order to create the role with the initial policy, the admin will have to create
the policy with broader permissions (for example, `arn:aws:s3:::*`).

### Permissions Boundaries and SCPs

Typically, when central admin teams restrict role creation it is because there is a need to
prevent developers from performing certain actions. [Permissions
boundaries](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html) can
be used to create security guardrails that set the maximum permissions that development teams can
have. For a deeper dive into permissions boundaries, read the blog post on [when and where to use
IAM permissions boundaries](https://aws.amazon.com/blogs/security/when-and-where-to-use-iam-permissions-boundaries/).

A permissions boundary is an [IAM customer managed
policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-vs-inline.html#customer-managed-policies)
and can be composed of individual policy statements. For a permissions boundary to be effective,
it must contain a set of base policy statements that prevent the entity from escaping out of the
boundary. These base policies include:

* Deny creating Roles/Users unless the same permissions boundary is attached. This prevents
  creating another Role that it can
  [assume](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use.html) which has
  administrator access.
* Deny modification/deletion of the permissions boundary itself
* Deny removing the permissions boundary attachment from the entity

In addition to the base policy, it must also contain customer specific policies that enforce the
organization’s guardrails. A common use case might be to allow developers to perform any actions,
except for actions that modify the VPC. In this case the admin can bootstrap the account with a
permissions boundary in place. At a high level the steps to enable permissions boundaries in CDK
are:

1. Create an IAM Managed Policy to serve as the Permissions Boundary.
2. Add the base policy to prevent privilege escalation
3. Add organization specific policy statements enforcing guardrails
4. Bootstrap the AWS environment specifying the Permissions Boundary to use
5. Apply the Permissions Boundary in any CDK applications deployed to the environment


An example policy to deny VPC actions might look something like this.

```yaml
PermissionsBoundary:
    Type: AWS::IAM::ManagedPolicy
    Properties:
      PolicyDocument:
        Statement:
          # ----- Begin base policy ---------------
          # If permission boundaries do not have an explicit allow
          # then the effect is deny
          - Sid: ExplicitAllowAll
            Action:
              - "*"
            Effect: Allow
            Resource: "*"
          # Default permissions to prevent privilege escelation
          - Sid: DenyAccessIfRequiredPermBoundaryIsNotBeingApplied
            Action:
              - iam:CreateUser
              - iam:CreateRole
              - iam:PutRolePermissionsBoundary
              - iam:PutUserPermissionsBoundary
            Condition:
              StringNotEquals:
                iam:PermissionsBoundary:
                  Fn::Sub: arn:${AWS::Partition}:iam::${AWS::AccountId}:policy/cdk-permissions-boundary
            Effect: Deny
            Resource: "*"
          - Sid: DenyPermBoundaryIAMPolicyAlteration
            Action:
              - iam:CreatePolicyVersion
              - iam:DeletePolicy
              - iam:DeletePolicyVersion
              - iam:SetDefaultPolicyVersion
            Effect: Deny
            Resource:
              Fn::Sub: arn:${AWS::Partition}:iam::${AWS::AccountId}:policy/cdk-permissions-boundary
          - Sid: DenyRemovalOfPermBoundaryFromAnyUserOrRole
            Action: 
              - iam:DeleteUserPermissionsBoundary
              - iam:DeleteRolePermissionsBoundary
            Effect: Deny
            Resource: "*"
          # ----- End base policy ---------------
          # -- Begin Custom Organization Policy --
          - Sid: DenyVPCNetworkModifications
            Action:
              - ec2:AssociateDhcpOptions
              - ec2:AssociateRouteTable
              - ec2:AssociateSubnetCidrBlock
              - ec2:AssociateVpcCidrBlock
              - ec2:AttachInternetGateway
              - ec2:AttachVpnGateway
              - ec2:CreateCustomerGateway
              - ec2:CreateDhcpOptions
              - ec2:CreateInstanceExportTask
              - ec2:CreateInternetGateway
              - ec2:CreateRoute
              - ec2:CreateRouteTable
              - ec2:CreateSubnet
              - ec2:CreateVpc
              - ec2:CreateVpcEndpoint
              - ec2:CreateVpcEndpointServiceConfiguration
              - ec2:CreateVpcPeeringConnection
              - ec2:CreateVpnConnection
              - ec2:CreateVpnConnectionRoute
              - ec2:CreateVpnGateway
              - ec2:DeleteCustomerGateway
              - ec2:DeleteDhcpOptions
              - ec2:DeleteEgressOnlyInternetGateway
              - ec2:DeleteInternetGateway
              - ec2:DeleteNatGateway
              - ec2:DeleteNetworkAcl
              - ec2:DeleteNetworkAclEntry
              - ec2:DeleteRoute
              - ec2:DeleteRouteTable
              - ec2:DeleteSubnet
              - ec2:DeleteVpc
              - ec2:DeleteVpcEndpointServiceConfigurations
              - ec2:DeleteVpcEndpoints
              - ec2:DeleteVpcPeeringConnection
              - ec2:DeleteVpnConnection
              - ec2:DeleteVpnConnectionRoute
              - ec2:DeleteVpnGateway
              - ec2:DetachInternetGateway
              - ec2:DetachVpnGateway
              - ec2:DisableVgwRoutePropagation
              - ec2:DisassociateRouteTable
              - ec2:DisassociateSubnetCidrBlock
              - ec2:DisassociateVpcCidrBlock
              - ec2:EnableVgwRoutePropagation
              - ec2:ModifySubnetAttribute
              - ec2:ModifyVpcAttribute
              - ec2:ModifyVpcEndpoint
              - ec2:ModifyVpcEndpointServiceConfiguration
              - ec2:ModifyVpcEndpointServicePermissions
              - ec2:ModifyVpcPeeringConnectionOptionsconnection
              - ec2:ReplaceRoute
              - ec2:ReplaceRouteTableAssociation
            Effect: Deny
            Resource: "*"
          # -- End Custom Organization Policy --
        Version: "2012-10-17"
      Description: "Bootstrap Permission Boundary"
      ManagedPolicyName: 
        Fn::Sub: cdk-permissions-boundary
      Path: /
```

#### Creation and Enforcement

The first three steps will create the permissions boundary and set the *enforcement* of the
permission boundary for any CDK applications that deploy to the bootstrap environment.

Supposing a Permissions Boundary was created with the name `cdk-permissions-boundary`, bootstrap
the environment specifying the `--custom-permissions-boundary` flag to attach the permissions
boundary to the CFN Execution Role.

```bash
cdk bootstrap --custom-permissions-boundary cdk-permissions-boundary
```

The bootstrap template will attach the permission boundary to the `cfn-exec` role which will
prevent CDK applications from deploying anything that modifies a VPC. As explained above, the
inclusion of the base policies also enforces that all IAM Roles and Users that are created by a
CDK application have the permissions boundary attached. This prevents a user from escalating their
access by creating a role with elevated privileges.

#### Applying the Permissions Boundary

Once the permissions boundary has been created and enforced, CDK applications will not be able to
create any IAM Roles or Users unless the permissions boundary is attached to the Role or User
being created. If an application attempts to create an IAM principal without the boundary the
CloudFormation deployment will fail with an error message like the below example.

```txt
API: iam:CreateRole User: arn:aws:sts::12345678912:assumed-role/cdk-custom-cfn-exec-role-12345678912-us-east-1/AWSCloudFormation
is not authorized to perform: iam:CreateRole on resource: arn:aws:iam::12345678912:role/my-stack-Role6C9272DF-1OBBVFUM03P7S
with an explicit deny in a permissions boundary
```

Attaching the permissions boundary to the Role will allow the CDK application to create the Role.

```ts
const stack = new Stack(app, 'MyStack');
new iam.Role(stack, 'Role', {
  assumeRolePolicy: new iam.ServicePrincipal('lambda.amazonaws.com',
  permissionsBoundary: iam.ManagedPolicy.fromManagedPolicyName(`cdk-${stack.synthesizer.bootstrapQualifier}-permissions-boundary-${stack.account}-${stack.region`);
});
```

The CDK also provides the ability to apply a permissions boundary to a CDK application globally. This can be done
either by adding the value to `cdk.json`.

```json
{
  "context": {
     "@aws-cdk/core:permissionsBoundary": {
       "name": "cdk-permissions-boundary"
     }
  }
}
```

Or directly in the `App` constructor.

```ts
new App({
  context: {
    [PERMISSIONS_BOUNDARY_CONTEXT_KEY]: {
      name: 'cdk-permissions-boundary',
    },
  },
});
```

It is also possible to enforce different permissions boundaries for different environments. For
example your CDK application may have the following `Stage`s:

* `DevStage` that deploys to a personal dev environment where you have elevated privileges
* `BetaStage` that deploys to a beta environment which has a more relaxed permissions boundary
* `GammaStage` that deploys to a gamma environment which has the prod permissions boundary
* `ProdStage` that deploys to the prod environment and has the prod permissions boundary

```ts
new Stage(app, 'DevStage');
new Stage(app, 'BetaStage', {
  permissionsBoundary: PermissionsBoundary.fromName('beta-permissions-boundary'),
});
new Stage(app, 'GammaStage', {
  permissionsBoundary: PermissionsBoundary.fromName('prod-permissions-boundary'),
});
new Stage(app, 'ProdStage', {
  permissionsBoundary: PermissionsBoundary.fromName('prod-permissions-boundary'),
});
```


This will attach the relevant permissions boundary to all IAM Roles and Users that are created as
part of each `Stage`.

