---
title: 'Serverless C# on AWS Lambda (pt. 2)'
date: 2017-02-06 19:45:26
tags:
---

*This is a multi-part series on using C#, AWS Lambda, and Serverless Framework to build REST services, ChatBots, Alexa Skills and more*

This week, we'll explore creating HTTP endpoints that execute your Lambda functions. I'll assume that you have the starter template from the [first post](/2017/02/05/Serverless-C-on-AWS-Lambda-pt-1/).

# Events and Triggers

Up until now, our function has lived in relative isolation. The only way we've been able to call the `hello` function has been using `serverless invoke` which translates to a direct AWS API call, or by using the AWS console to test the function directly. Unsurprisingly, these are not the intended interaction models for AWS Lambda functions.

In AWS Lambda services, your functions will typically execute in response to various types of events. These can be things such as "a file was uploaded to S3", "a server crashed", "an item was added to your DynamoDB database", or even "an HTTP request was made to this API Gateway endpoint". In this post, you'll see how you can configure respond to the latter, an HTTP endpoint.

# API Gateway

Because Lambda functions can be triggered by many types of events, it's worthwhile to highlight that neither the AWS Lambda infrastructure nor your function are aware of networking or HTTP. Functions simply take in a stream of data and output a stream of data (though as we'll see, serialization plays a part in simplifying this interface). The event sources determine the schema of the input data they provide to the function as well as the output schema they expect back from the function (if any).

The event source used for responding to HTTP events is AWS's API Gateway. API Gateway is a service for exposing HTTP endpoints that map to any number of your AWS resources that you want those endpoints to execute. In our case, we'll be triggering AWS Lambda functions and returning their output.

The lifecycle of a web request looks roughly like this:

1. A client makes a `GET` request to `/foo` which is configured a API Gateway endpoint
2. The `GET` method for the `/foo` resource is mapped to the `hello` Lambda function
3. API Gateway wraps up data about the `GET` request in a json payload which contains the headers, HTTP method, full request path, request body, and other HTTP metadata
4. The `hello` function parses that input and decides how to respond
5. The `hello` function returns a json paylod with the status code, headers, and the response body back to API Gateway
6. API Gateway formats that response and returns it as a proper HTTP resopse back to the client

Using Serverless Framework, we won't have to worry about many of the particulars of this flow.

# Configuring the Event

In our `serverless.yml`, we have the following for our function configuration:

```yaml
functions:
  hello:
    handler: CsharpHandlers::AwsDotnetCsharp.Handler::Hello
```

What we're going to add to his is the `events` property for our `hello` function. Change this section to look like the following:

```yaml
functions:
  hello:
    handler: CsharpHandlers::AwsDotnetCsharp.Handler::Hello
    events:
      - http: GET foo
```

That's it!

# Testing the Event (Take #1)

Now that we've modified our service definition (`serverless.yml`), let's remember what we need to do to deploy it. Note that since we haven't made modifications to the C#, we don't need to build an updated package.

```shell
serverless deploy
```

When that command completes, you should see some info in the "Service Information" output that wasn't populated before (your subdomain will be different):

```yaml
endpoints:
  GET - https://pya6b2f8i0.execute-api.us-east-1.amazonaws.com/dev/foo
```

