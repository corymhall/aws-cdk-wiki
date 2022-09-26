When you execute the CDK CLI, you may see a message like this:

```
NOTICES

12345	Toggling off auto_delete_objects for Bucket empties the bucket

	Overview: If a stack is deployed with an S3 bucket with
	          auto_delete_objects=True, and then re-deployed with
	          auto_delete_objects=False, all the objects in the bucket
	          will be deleted.

	Affected versions: cli: <=1.126.0

	More information at: https://github.com/aws/aws-cdk/issues/12345

If you don’t want to see a notice anymore, use "cdk acknowledge ". For example, "cdk acknowledge 12345".
```

We call these messages "notices". They are a mechanism for the CDK team to push information to you, without having to send you an email for every AWS account you use. Plus, the email we would have sent might not reach the right person.

We put out a notice whenever we think the message is so important that everyone who might be affected needs to see it.

## What should I do when I see a notice?

The first thing you should do is determine whether you are affected by the described issue. We do our best to display notices only to affected customers (based on the version of the framework or CLI you are using), but sometimes that is not possible, and you will need to review the notice text and potentially the linked GitHub issue to know whether or not you are affected. For example, there might be a bug that is only triggered if you use a certain construct a certain way, and we can't look at your code to determine that.

**If you are affected**, you should upgrade to the next available version that is no longer affected by the notice. The CLI and framework each have an independent version number. It might be that no fixed version is available yet. In that case you can consider downgrading, or waiting until the fixed version is released. Whenever we put out a notice, we are working to release a fixed version as soon as possible.

**If you determine that you are not affected**, you can silence the notice by running the `cdk acknowledge <number>` command. This will stop the notice from being displayed for the current project.

## Frequently Asked Questions

Here are the answers to some more questions we occasionally get:

## What if I see the error "Could not refresh notices" in my verbose log?

This indicates we cannot contact the internet to download the latest set of notices. If you are seeing this, you are probably running the CLI in a network-restricted environment. 

This error will not affect the normal operation of the CLI. If you are investigating a failure, keep on reading the log.

## What if I don't want notices?

Run the CDK CLI with `--no-notices`, `CDK_NOTICES=false` or put `{ "notices": false }` in `cdk.json`.
