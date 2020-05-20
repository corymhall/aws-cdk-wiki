Canned responses to GitHub issues.

Some questions are asked over and over again, and typing out the same answers over and over again gets tiresome.

## Guidance question on the bug tracker

*(tag as **Guidance**, and close)*

> We're happy to help you out, but GitHub issues is not a support forum. There are websites much better suited for that, such as StackOverflow.
>
> Please ask that same question again on \[StackOverflow under the aws-cdk](https://stackoverflow.com/questions/ask?tags=aws-cdk) tag. If you can't get a response to your question, please bring it to our attention again and we will do our best to answer it.

## Mixing L1 and L2

*(tag as **cause/mixing-l1-and-l2**, and probably also as **cause/python-no-types**)*

> Looks like you are mixing \[CloudFormation Resource classes with Construct Library Classes\](https://docs.aws.amazon.com/cdk/latest/guide/constructs.html#constructs_lib). 
>
> Classes and interfaces that start with the name `Cfn` are CloudFormation Resources classes (also called L1 classes), which map directly onto CloudFormation. They generally only work in primitives, so they'll accept strings and numbers and structs of strings and numbers, but never other classes. For example, classes at the L1 level will take a Log Group ARN as a string, not a LogGroup object.
>
> Classes and interfaces that do not start with `Cfn` are the AWS Construct Library classes (also called L2 classes), which are higher level classes written by hand, which have more abstractions and features written in them. They generally favor taking other L2 resources as object references, instead of accepting strings and numbers. For example, classes at the L2 level will take a `LogGroup` object, not an ARN string.
>
> Passing an L1 class to an L2 construct, or an L2 class to an L1 construct, will never work. Most of the time your type checker will warn you that you are passing the wrong object, but if you are using Python you may see this as properties not showing up where you expect them to.
>
> GitHub issues is not a support forum. If you need support to achieve what you want to do, please ask for support on \[StackOverflow under the aws-cdk](https://stackoverflow.com/questions/ask?tags=aws-cdk) tag. If you can't get a response to your question there, please bring the StackOverflow post to our attention again and we will do our best to answer it there.