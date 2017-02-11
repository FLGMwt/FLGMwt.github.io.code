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

1. A client makes a `GET` request to `/helloworld` which is configured a API Gateway endpoint
2. The `GET` method for the `/helloworld` resource is mapped to the `hello` Lambda function
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
      - http: GET helloworld
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
  GET - https://pya6b2f8i0.execute-api.us-east-1.amazonaws.com/dev/helloworld
```

Neat! Here we see the API Gateway endpoint that `serverless` generated which should map to your function. Copy your URL (not the one posted above) and navigate to it in your browser. You'll likely see the following:

```json
{"message": "Internal server error"}
```

What gives? Well, the good news is that we didn't get a broswer error page, so we're probably doing *something* right. Maybe the problem is with our function? Let's see if our function works in API Gateway.

# API Gateway console

Since our HTTP resource, `/dev/helloworld`, is managed by API Gateway, we can look for a point of failure there. Log in to the AWS Console, and navigate to the "API Gateway" service.

In API Gateway, you'll see `dev-foo` listed under the `APIs` section. An "API" groups all the endpoints of your serverless service. If you click on `dev-foo`, you'll see that we have one, `/helloworld` which supports the `GET` HTTP method. Clicking on `GET` under `/helloworld` brings us to the pipeline diagram for that specific action. It describes the request/resposne lifecycle of our endpoint:

1. A client makes a HTTP request
2. The request approaches the configured integration and is transformed to the format the integration prefers
 - An API integration is the thing that actually handles the request, in our case, a Lambda function
3. The integration receives the transformed request and performs some logic with it
4. The integration returns a response to API Gateway
5. The integration's response is transformed into a HTTP response
6. The client receives the HTTP response

There's a lot of opportunities for misconfiguration in that pipeline. Thankfully, API Gateway makes it easy to test resources. Click on "Test" on the "Client" box on the left. You'll see a page that allows you to try out making requests to your resource with different query strings, headers, and request bodies (for supported HTTP verbs).

Since we're not doing anything special in our function with query strings or headers yet, leave everything blank and click "Test".

On the right side of the screen, you'll see that we still the get the `"Internal server error"` message, but more helpfully, we have logs for all parts of the request pipeline.

First, there's a lot of information about the request, such as our empty body, headers and the like. Then, it shows the `body after transformation` which is the full batch of data which our Lambda function receives.

Surprisingly, we then see the response body we expected with the `Go Serverless` message. Looks like our function didn't fail after all. What *did* fail is the next step. A couple lines down, we see the following message:

```
Execution failed due to configuration error: Malformed Lambda proxy response
```

Hrm. Looks like we didn't format the output of our Lambda function the way that API Gateway was expecting. Diving deep in to the [AWS documentation](http://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-set-up-simple-proxy.html#api-gateway-simple-proxy-for-lambda-output-format), it was expecting a resposne of the following format:

```json
{
    "statusCode": httpStatusCode,
    "headers": { "headerName": "headerValue", ... },
    "body": "..."
}
```

# Formatting the Function resposne





# Serverless logs

Even though functions live in contained execution environments, that doesn't mean you can't access application logs or exception messages from them. Standard output from Lambdas get fed into CloudWatch, the standard logging collector for all of AWS.

We can use the `serverless` CLI to find logs specific to our function. Try the following:

```bash
serverless logs -f hello
```


Todo: before sls logs, jump into API Gateway, show test request.