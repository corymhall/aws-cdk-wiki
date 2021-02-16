## Applications 

CDK **Applications** should use point versions of Core CDK libraries. Specifically, they should NOT be using **range** dependencies. So:

```js
{ 
  "dependencies": {
    "@aws-cdk/core": "1.80.0",   /* NOT: "^1.80.0" */
  }
}
```

And all versions should be the same. This is necessary because it is required that only exactly *one* copy of the CDK libraries exist in your dependency tree, and having all versions exactly the same is the only way to get NPM to deduplicate libraries properly.

CDK itself *should* have used `peerDependencies` to encode dependencies between libraries, and if we had this wouldn't have been an issue. However, at the time we made the decision to use `dependencies` (instead of `peerDependencies`), this was necessary to get the best user experience, given the behavior of NPM of versions lower than 7.

## Libraries

We will rectify our own mistakes with the CDK 2.x version line.

In the mean time, don't make the same mistake we did! Use CDK as a `peerDependency` instead of a regular `dependency`, this will make it possible to use your library with any version of the CDK, rather than having to re-release your library every time the CDK releases a new version.

Don't use experimental APIs in your library (see the documentation to confirm), then set up your `package.json` like this:

```js
{
  "peerDependencies": {
    "@aws-cdk/core": "^1.80.0",  // <-- this is the MINIMUM version you support
    "@aws-cdk/aws-foo": "^1.80.0", 
  },
  "devDependencies": {
    "@aws-cdk/core": "1.80.0",   // <-- purposely no ^ here to make sure you test against the minimum version
    "@aws-cdk/aws-foo": "1.80.0", 
  }
}
```

You then need to give the following guidance to your *consumers*:

* If your consumer is using NPM7: no additional work needed (NPM7 will automatically install `peerDependencies`).
* If your consumer is using NPM6 or older: they MUST add your peerDependencies (at their desired version) to their own dependencies:

```js
/* application's package.json */
{
  "dependencies": {
    "awesome-library": "^1.0.0",
    "@aws-cdk/core": "1.90.0",   // <-- this can be higher. Still no ^
    "@aws-cdk/aws-foo": "1.90.0" // <-- you need to add this because of 'awesome-library'
  }
}
```
  