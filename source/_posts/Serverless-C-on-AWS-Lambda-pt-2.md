---
title: 'Serverless C# on AWS Lambda (pt. 2)'
date: 2017-02-11 14:45:26
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
5. The `hello` function returns a json payload with the status code, headers, and the response body back to API Gateway
6. API Gateway formats that response and returns it as a proper HTTP response back to the client

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

What gives? Well, the good news is that we didn't get a browser error page, so we're probably doing *something* right. Maybe the problem is with our function? Let's see if our function works in API Gateway.

# API Gateway console

Since our HTTP resource, `/dev/helloworld`, is managed by API Gateway, we can look for a point of failure there. Log in to the AWS Console, and navigate to the "API Gateway" service.

In API Gateway, you'll see `dev-foo` listed under the `APIs` section. An "API" groups all the endpoints of your serverless service. If you click on `dev-foo`, you'll see that we have one, `/helloworld` which supports the `GET` HTTP method. Clicking on `GET` under `/helloworld` brings us to the pipeline diagram for that specific action. It describes the request/response lifecycle of our endpoint:

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

Hrm. Looks like we didn't format the output of our Lambda function the way that API Gateway was expecting. Diving deep in to the [AWS documentation](http://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-set-up-simple-proxy.html#api-gateway-simple-proxy-for-lambda-output-format), it was expecting a response of the following format:

```json
{
    "statusCode": httpStatusCode,
    "headers": { "headerName": "headerValue", ... },
    "body": "..."
}
```

# Formatting the Function Response

To model the function output the way that API Gateway expects, we'll need to start making some code changes. Yay!

Delete the existing `Response` class and add the following class:

```csharp
// The following usings will be required:
using System.Collections.Generic;
using Newtonsoft.Json;

// ... later

public class Response
{
    [JsonProperty("statusCode")]
    public int StatusCode { get; set; }

    [JsonProperty("headers")]
    public Dictionary<string, string> Headers { get; set; }

    [JsonProperty("body")]
    public string Body { get; set; }
}
```

Here we see the three required properties represented: "statusCode", "headers", and "body".

## `JsonProperty` and Serialization
One thing to clarify is the use of the `JsonProperty` attribute. API Gateway expects `camelCase` for all property names. However, idiomatic C# uses `PascalCase` for public properties. Because the Lambda Serializer is based on [Json.NET](http://www.newtonsoft.com/json), we can use `JsonProperty` tell the serializer to do two things:

- When serializing the C# object with the `Body` property, name the json property `body`
- When deserializing a json object, look for the property `body` and use it to populate the `Body` property in C#

In a future post, we'll go into more detail about serialization and look at easier ways to handle the mapping between idiomatic C# and idiomatic json.

## Testing the New Response

To set the new response model, modify the `Hello` method as follows:

```csharp
public Response Hello(Request request)
{
    return new Response
    {
        StatusCode = 200,
        Headers = new Dictionary<string, string>(),
        Body = "Go Serverless! Hello from HTTP",
    };
}
```

Jump back to the command line and build and deploy the modified function:

```shell
./build.ps1 # or ./build.sh
serverless deploy
```

Try the endpoint URL in the browser again. You should now see the `Go Serverless! Hello from HTTP` message. Yay!

But wait, what happened to the structure of our original message?

# Complex Response Body

I cheated a little bit in the last code change to get the HTTP request working. Let's get back to the original response data schema. Add the following class below the definition of `Response`.

```csharp
public class ResponseBody
{
    public string Message { get; set; }
    public Request Request { get; set; }
}
```

Also modify the `Hello` method again:

```csharp
public Response Hello(Request request)
{
    var responseBody = new ResponseBody
    {
        Message = "Go Serverless! Hello from HTTP!",
        Request = request,
    };
    return new Response
    {
        StatusCode = 200,
        Headers = new Dictionary<string, string>(),
        Body = JsonConvert.SerializeObject(responseBody),
    };
}
```

What's going on here? Well, remember that the `body` property of the response must be of type `string`? That means that we have to explicitly serialize our response body as a string.

After another `./build.ps1` and `serverless deploy`, try your function in the browser again and you should get the following (though not as nicely formatted):

```json
{
  "Message": "Go Serverless! Thanks for reading the first part of this series!",
  "Request": {
    "Key1": null,
    "Key2": null,
    "Key3": null
  }
}
```

# Using Request Data

Neat! Now we know how to massage and structure our responses so that API Gateway can handle them. Since the move to using HTTP events, though, our example `Request` type hasn't made a lot of sense. The data that comes through from the browser's request and the API Gateway transformation doesn't have any of `Key1`, `Key2`, or `Key3`, so those properties aren't populated when Lambda attempts to serialize them.

So what data *can* we use in our Lambda function? Since `GET` requests don't have request bodies, let's use query parameters.

What will that look like? Instead of digging though the deep waters of AWS documentation, recall that the API Gateway resource test page logged what the transformed integration input looked like. If we add a query string to the test input, we can see what or function would get.

Go to the API Gateway test page again and put `name=Ada` in the "Query String" field, then click "Test". Take a look at the output labeled "Endpoint request body after transformations":

```json
{
  "resource": "/foo",
  "path": "/foo",
  "httpMethod": "GET",
  "headers": null,
  "queryStringParameters": {
    "name": "ada"
  },
  // ...
}
```

From this, it looks like it'll be pretty straightforward to create a new `Request` class that models the integration request. Replace the exising `Request` class with the following:

```csharp
public class Request
{
    [JsonProperty("httpMethod")]
    public string HttpMethod { get; set; }

    [JsonProperty("queryStringParameters")]
    public Dictionary<string, string> QueryStringParameters { get; set; }
}
```

Note that we don't have to have a property for everything we receive. All other properties will be silently ignored. With this `Request` class in place, modify the `Hello` method one last time to look like this:

```csharp
public Response Hello(Request request)
{
    var name = "Jane Doe";
    if (request.QueryStringParameters != null
        && request.QueryStringParameters.ContainsKey("name"))
    {
        name = request.QueryStringParameters["name"];
    }
    var responseBody = new ResponseBody
    {
        Message = $"Go Serverless! Hello {name} from HTTP!",
        Request = request,
    };
    return new Response
    {
        StatusCode = 200,
        Headers = new Dictionary<string, string>(),
        Body = JsonConvert.SerializeObject(responseBody),
    };
}
```

We want to default the name and not error if one is not provided. We saw earlier that `queryStringParameters` is null if there is no query string, so we need to check against that.

After one last `./build.ps1` and `serverless deploy`, try your function out. You can test it like we were before, or you can add `?name=Ada`, and you'll see a response like this:

```json
{
  "Message": "Go Serverless! Hello ada from HTTP!",
  "Request": {
    "method": "GET",
    "queryStringParameters": {
      "name": "ada"
    }
  }
}
```

Once again, well done! Next post, we'll extend our function to handle HTTP POSTs and build a fully-functioning chat bot!