# AWS SAM — Concepts

What SAM is, why it exists, and the mental model behind a SAM template.
Pure concept — no commands here (see the SAM commands reference for those).

Philosophy: rather than a theoretical summary of every possible option (there
are hundreds, nobody memorizes them all), this covers a solid understanding
of what gets written and why. For the rest, knowing how to read the official
docs efficiently is the real skill.


## What SAM actually is

SAM (Serverless Application Model) is a framework built on top of
CloudFormation. A SAM template is a CloudFormation template with extra
shorthand syntax (`AWS::Serverless::*`) specifically for serverless
resources — Lambda functions, APIs, simple tables. At deploy time, SAM
**transforms** those shortcuts into their full, verbose CloudFormation
equivalent before anything actually gets created.

This is why every SAM template starts with:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
```

`Transform: AWS::Serverless-2016-10-31` is the line that activates SAM — it
tells CloudFormation "translate the `AWS::Serverless::*` shortcuts in this
file into standard CloudFormation resources before deploying." Without this
line, the file would be a plain CloudFormation template, not SAM.

A SAM project also combines two things that stay separate conceptually: the
**template** (declarative — "I want a Lambda, a DynamoDB table, an API
endpoint") and the **application code** (imperative — "when the Lambda
receives a request, do X"). SAM orchestrates packaging and deploying both
together.


## SAM vs raw CloudFormation

SAM is not a replacement for CloudFormation — it's a layer on top of it.
Every SAM template ultimately becomes a CloudFormation template
(`Transform` does that translation). The trade-off:

- **Raw CloudFormation** — verbose but fully explicit; every property of every resource is spelled out. No hidden shortcuts.
- **SAM** — shorthand for serverless-specific patterns (`AWS::Serverless::Function`, `Events`, `Policies` templates) that would otherwise take many more lines of raw CloudFormation to express. Less typing, same underlying mechanism.

Resources that aren't serverless-specific (a DynamoDB table, an S3 bucket)
are written the same way in both — SAM doesn't shorten those; it only adds
shortcuts for the serverless-specific pieces (functions, APIs, simple
tables).


## SAM vs Terraform — same family, different scope

Both are Infrastructure as Code, but they don't cover the same ground:

```
Terraform  → describes infrastructure ONLY (no concept of "the application code")
             → the Lambda's code exists separately; Terraform just references the zip/artifact

SAM        → describes infrastructure AND packages/deploys the code together
             → `build` compiles the code + dependencies
             → `deploy` ships infra + code in one operation
```

SAM is "serverless-specialized": it natively understands the
build → package → deploy cycle of a Lambda function, while Terraform stays
general-purpose and stays neutral about application content — it doesn't
know or care what's inside the artifact it's pointing to.



## Anatomy of a SAM template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  # Default values shared across functions (e.g. Runtime, Timeout)

Resources:
  # Every AWS resource actually deployed (tables, functions, APIs...)

Outputs:
  # Useful values shown after deployment (e.g. the API's URL)
```

- **`AWSTemplateFormatVersion`** — the CloudFormation format version; a fixed value by historical convention, not something you change.
- **`Transform: AWS::Serverless-2016-10-31`** — activates SAM (see above).
- **`Resources`** — the core of the file; covered in detail below.
- **`Outputs`** — optional, useful for exposing values (URLs, ARNs) after a deployment finishes.



## `Resources` — everything deployed, not just functions

`Resources` contains **every AWS resource physically created** at deploy
time — a Lambda function is one type of resource among many others (same
category as a database table, a storage bucket, a queue, a topic, a user
pool...).

```yaml
Resources:
  SomeTable:
    Type: AWS::DynamoDB::Table        # a database
    Properties: ...

  SomeFunction:
    Type: AWS::Serverless::Function    # a Lambda
    Properties: ...

  SomeBucket:
    Type: AWS::S3::Bucket              # storage
    Properties: ...
```

"Resource" is the generic cloud-computing term: any object managed by the
provider (a database, a server, a file, a function, a virtual network...).
A "Lambda function" is just one resource type among hundreds of possible
ones.



## The general pattern of a resource

Every resource, regardless of its type, follows the same shape:

```yaml
Resources:
  LogicalResourceName:            # identifier used to reference it elsewhere in the template
    Type: AWS::Service::Resource   # the exact AWS resource type
    Properties:
      PropertyName: value          # properties specific to that resource type
```

The **logical name** (`LogicalResourceName`) is not necessarily the actual
name of the resource on AWS — it's an identifier internal to the template,
used for cross-references (`!Ref`, `!GetAtt`).

**Concrete example**, from an actual template:

```yaml
HelloWorldFunction:
  Type: AWS::Serverless::Function
  Properties:
    CodeUri: hello_world/
    Handler: app.lambda_handler
    Runtime: python3.14
    Architectures:
      - x86_64
    Events:
      HelloWorld:
        Type: Api
        Properties:
          Path: /hello
          Method: get
```

`HelloWorldFunction` is the logical name; `AWS::Serverless::Function` is the
type; everything under `Properties` configures that specific function
(where the code lives, the handler entry point, the runtime, and — via
`Events` — how it gets triggered).



## Cross-references between resources (intrinsic functions)

Two intrinsic functions come up constantly:

- **`!Ref LogicalName`** — returns the resource's primary identifier (what exactly depends on the resource type: a table name for DynamoDB, an ARN for some other types...).
- **`!GetAtt LogicalName.Attribute`** — returns a specific attribute of the resource (e.g. `!GetAtt SomeFunction.Arn`).

A third, `!Sub`, is common in `Outputs` for building a string that embeds
one or more references or pseudo-parameters:

```yaml
Outputs:
  HelloWorldApi:
    Description: API Gateway endpoint URL for Prod stage for Hello World function
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.${AWS::URLSuffix}/Prod/hello/"
  HelloWorldFunction:
    Description: Hello World Lambda Function ARN
    Value: !GetAtt HelloWorldFunction.Arn
```

Here, `ServerlessRestApi` is an **implicit** resource — SAM automatically
creates an API Gateway behind the scenes whenever a function declares an
`Events: Type: Api`, without it ever being written explicitly under
`Resources`. `${AWS::Region}` and `${AWS::URLSuffix}` are AWS
pseudo-parameters, resolved automatically at deploy time (current region,
current partition's URL suffix).



## One template vs several (nested stacks)

CloudFormation allows splitting infrastructure across multiple templates
via **nested stacks** (`AWS::CloudFormation::Stack`), where a root template
references sub-templates that each deploy their own stack. This is mainly
useful for very large projects or multiple teams — the added complexity of
managing dependencies between stacks isn't justified below that scale.

For a single-developer or small-scale project, a **single `template.yaml`**
that grows over time, organized into clearly commented sections, is
generally the more practical and readable choice — easier for anyone
reviewing the repo to follow end to end, and consistent with how most
introductory serverless labs and courses are structured (no nested stacks).



## References

- [SAM template anatomy](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-specification-template-anatomy.html)
- [SAM Globals section](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-specification-template-anatomy-globals.html)
- [CloudFormation intrinsic function reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html)
- [Nested applications](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/using-nested-applications.html)
- [AWS::Serverless::Function reference](https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md#awsserverlessfunction)