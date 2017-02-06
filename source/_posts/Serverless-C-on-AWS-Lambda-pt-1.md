---
title: 'Serverless C# on AWS Lambda (pt. 1)'
date: 2017-02-05 15:34:19
tags:
---

*This is a multi-part series on using C#, AWS Lambda, and Serverless Framework to build REST services, ChatBots, Alexa Skills and more*

You might have heard that back in December, [https://aws.amazon.com/blogs/compute/announcing-c-sharp-support-for-aws-lambda/](AWS Lambda added support for C#). Welcome to the club, fellow .NET developer! AWS Lambda is the logical conclusion of the move from Infrastructure-as-a-Service to Platform-as-a-Service to Containers-as-a-Service to what we're going to be dealing with: Functions-as-a-Service (often referred to as Serverless).

Serverless applications offer a number of benefits such as ease of deployment, effortless scalability, and cost elasticity (you only pay for when your service is used). Because the pros and cons of serverless applications have been discussed at length, I'll leave the analysis to [those much smarter than I](https://martinfowler.com/articles/serverless.html) and stick to a guide.

That said, if your functions are itching to get up in the cloud, let's get started.

# Setup

First, we'll need the [Serverless Framework](https://serverless.com/) tools which will help us create and deploy functions. You can install it via npm as follows (if you don't have node or npm, [install them here](https://nodejs.org/en/):

``` shell
npm install --global serverless
```

**Note**: While it's certainly possible to manage Lambda functions and applications using just the console or raw [CloudFormation](https://aws.amazon.com/cloudformation/), Serverless Framework is a popular option for managing and deploying functions, not least of all because it simplifies management of infrastructure as code. Throughout this series, we'll see the power of that.

Before we can do much with `serverless`, we need to configure the credentials for the AWS user that will be managing functions, S3 objects, and other AWS resources on our behalf. The steps for creating one are as follows:

1. Log in to your AWS console
2. Navigate to the IAM service
  - Identity and Access Management (IAM) is where you manage users, groups, roles, and policies (permissions of sorts) for your AWS account
3. Click "Users" and then "Add user"
4. Give your user a name, such as `serverless-tutorial`
5. For access type, check "Programmatic access"
6. On the "Permissions" step, select, "Attach existing policies directly"
7. Search for and select the "AdministratorAccess" policy
8. Click "Next: Review" and then "Create User"
9. Your user is now created but **don't leave this page yet**

Your new user has an access key and a secret key associated with it that are used to authenticate and authorize actions on your account. You're only able to access the secret key on this page and if you don't grab it now, you'll have to generate a new set of keys.

Now we're going to copy both keys into a serverless command that sets up your new credentials on your machine:

``` shell
serverless config credentials --provider aws --key <YOUR_ACCESS_KEY> --secret <YOUR_SECRET_KEY>
```

Great! The `serverless` command now can act on behalf of the `serverless-tutorial` user we created and we're able to get started creating and deploying functions.

# Your First Function

The `serverless` command allows you to create functions from templates for the different languages that AWS Lambda supports, such as nodejs, python, Java, Scala, and our friend C#. Create a new C# Serverless project like this:

``` shell
serverless create --template aws-csharp --path serverless-tutorial
```

A new directory named `serverless-tutorial` has been created with some interesting files in it. `cd` there and we'll take a look at each one.

## C# Serverless Project Structure

### `serverless.yml`

This file contains all of the metadata about your Serverless service so that the `serverless` command knows how to wire everything up in AWS. You'll notice that the provided template has a lot of commented out examples. If we pull those out, we'll see that the default template provides this for you:

```yaml
service: serverless-tutorial
```

This is simply the name of your service. Since we provided `serverless-tutorial` as the terminal directory in the `--path` argument of `serverless create`, our service is named that by default.

```yaml
provider:
  name: aws
  runtime: dotnetcore1.0
```

This block declares the cloud provider as AWS. Serverles Framework currently only supports AWS, but since many cloud providers offer similar serverless options, this could expand.

The `runtime` property states that the runtime for our function will be .NET Core 1.0. AWS Lambda supports NodeJS, Python, JVM-based languages, and .NET Core-based languages.

```yaml
package:
 artifact: bin/release/netcoreapp1.0/publish/deploy-package.zip
```

The `artifact` property of the `package` block determines the path to our bundled function code on our local workstation. We'll see later how the build scripts generate this package. On deploy, this file will be published to AWS S3 and the Lambda will be pointed at it.

```yaml
functions:
  hello:
    handler: CsharpHandlers::AwsDotnetCsharp.Handler::Hello
```

The `functions` block lists all of our functions. We only have one created by the template: the `hello` function.

With the `handler` property, each function points to a specific method in our C# code which will take the initial input to the Lambda and return the final output from the Lambda. The syntax for this value is as follows: `{assembly name}::{namespace}.{class name}  ::{method name}`

Taken altogether, when this serverless is deployed, the `serverless.yml` instructs the `serverless` CLI to create a stack called `serverless-tutorial`, which is on the .NET Core runtime. The service's code will live in a package located at `bin/Release/netcoreapp1.0/publish/deploy-package.zip` on the local workstation which will be pushed to AWS S3. Included in this package is the C# method `Hello` in the `AwsDotnetCsharp.Handler` namespace which will handle all calls to the `hello` AWS Lambda function.

### `Handler.cs`

This C# code file contains the handler for our first function, the `Hello` method. Most importantly, we see that it takes in a single object of type `Request` which is defined as a POCO later in the file. It's only line of implementation is to return an object of type `Response` which is also defined later in the file which takes a `string` message and an echo of the request.

Something to note here is that there is no `System.Web` reference nor any semblance of a framework. A serverless function's job is to very directly take in raw data and return raw data.

### `AssemblyInfo.cs`

This file provides an attribute for the assembly as a whole, defining the serializer class that should be used for deserializing incoming streams and deserializing outgoing data into streams.

You shouldn't need to worry about this too much, we'll be revisiting it in the REST function post.

### `build.sh` and `build.ps1`

These files handle the packaging of our code in preparation for `serverless` deploying it.

Each file does the following:

- restores packages
- publishes the project to the `bin/release/netcoreapp1.0/publish` directory
- zips up the build artifacts as `deploy-package.zip`

### `project.json`

This is the .NET Core project file which contains information about the C# code such as:

- package metadata (assembly name, version, etc.)
- dependencies
- target frameworks

We don't have to worry about this much until the next post.

## Deploying Our Function

By now, we know enough about what's included in the default `aws-csharp` Serverless Framework template. Let's build and deploy it and see what happens!

Run the following:

```shell
./build.ps1
serverless deploy
```
or (depending on your system)
```shell
./build.sh
serverless deploy
```

After a few minutes, you should get some output like this:

```
Service Information
service: serverless-tutorial
stage: dev
region: us-east-1
api keys:
  None
endpoints:
  None
functions:
  serverless-tutorial-dev-hello: arn:aws:lambda:us-east-1:1937231523426:function:serverless-tutorial-dev-hello
```

This means your deploy was successful! Also note that you can get this same output at any time by running `serverless info`. If you log into your AWS console, you should see your new function in the functions list. If you don't, make sure you select the same region in the dropdown at the top right of your console as the region listed in the `Service Information` output.

## Testing Our Function

Now that we've got some code, let's see it in action. Run the following:

```shell
serverless invoke -f hello
```

This sends a body-less request to our function. The output will look something like this:

```json
{
    "Message": "Go Serverless v1.0! Your function executed successfully!",
    "Request": {
        "Key1": null,
        "Key2": null,
        "Key3": null
    }
}
```

Neat! We know that `Hello` returned a `Response` object with the property `Message` which was assigned with the default string in `Hello`'s call to `Response`'s constructor. We didn't provide a request body with properties `Key1`, `Key2`, or `Key3`, so those items are null.

## Customizing the Response

We got the default template running, but that's not a great deal of fun. Try changing the message string in `Handler` to a message of your own:

```csharp
return new Response("Thanks for reading the first part of this series!", request);
```

Remember, `serverless` deploys our code from `deploy-package.zip`, but since we changed the source, we need to rebuild the package. Again, run:

```shell
./build.ps1
serverless deploy
```
or (depending on your system)
```shell
./build.sh
serverless deploy
```

Now try `serverless invoke -f hello` and you should see your custom message:

```json
{
    "Message": "Thanks for reading the first part of this series!",
    "Request": {
        "Key1": null,
        "Key2": null,
        "Key3": null
    }
}
```

Well done! Next post, I'll cover triggering your function with an HTTP request. Microservices, here we come!