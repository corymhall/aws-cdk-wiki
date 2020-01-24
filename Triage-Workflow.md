This wiki includes a bunch of canned responses for GitHub issues. Feel free to edit.

### Bugs

Bugs are triaged for priority:

- **P0**: Catastrophic failure. Prevents significant usage for many/most customers.
- **P1**: Severe bug, shared by multiple customers. Blocks usage and/or workaround is too complex.
- **P2**: Small/medium bug. Only affects edge-case or tertiary code area and/or has a decent workaround.

### Features

Features are triaged as follows:

 - Determine if there is a duplicate or if it is related to some other feature/module/area, and close the issue.
 - Propose that the feature request will be submitted as an RFC to the [CDK RFC repo](https://github.com/aws/aws-cdk-rfcs), and transfer the issue to the RFC repo.
 - Estimate the size of the feature S/M/L and label it `effort/small`, `effort/medium`, `effort/large`. This estimation can serve the construct library planning work in the next quarter.

Canned responses:

- Transfer to RFC repo: `This is a very interesting idea. I am transferring this to the RFC repo. Please follow the [RFC Repo README](https://github.com/aws/aws-cdk-rfcs/README.md) in order to submit this as an RFC.`

### Questions/Guidance

For *guidance* issues, in the most part either the owner can simply answer the question or kick off a discussion, or preferably refer the user to Stack Overflow to ask their question there. There is tons of value for these questions to be archived in SO over GitHub and in many cases the community will be able to answer.

```
We're happy to help you out, but GitHub issues is not a support forum. There are
websites much better suited for that, such as StackOverflow.

Please ask that same question again on [StackOverflow under the
aws-cdk](https://stackoverflow.com/questions/ask?tags=aws-cdk). If you can't get
a response to your question, please bring it to our attention again and we will
do our best to answer it.
```