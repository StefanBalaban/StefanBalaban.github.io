---
layout: post
title:  "Developing a OneDrive Backup Service With .NET 6: Part 3 Graph API"
date:   2022-04-07 21:02:23 +0100
categories: C# .NET 
---
In this post I'll cover using the Graph API SDK with a customized HttpClient to keep authenticated.

# OAuth 2.0 Flow
Initially I planned to just use an Azure AD App Registration and have the Worker Service use the client credentials flow to generate a token every time it starts the backup process. However, as the personal OneDrive can only be accessed through the /me endpoint, I had to implement a delegated flow which can only be done with auth code.
I considered a few different ways of doing this but in the end I decided of creating a small web app that will allow me to complete the OAuth 2.0 flow and then store the access and refresh token within its memory. It would have an API endpoint which would retrieve the token and, internally, refresh a stale token. I termed this app "The Session Gateway".

Using Minimal API's (and omitting the basic error handling of returning 500 errors in the pasted code example) the bulk of the application fit inside of the Program.cs file:
```csharp
app.MapGet("/authorize", () =>  
{
    // Request endpoint is the Microsft auth endpoint
    // Along with all the query params like tenantId...
    return Results.Redirect(config.RequestEndpoint);  
});  
  
app.MapGet("/access-token", async (AuthenticationService authenticationService) =>  
{
    return Results.Ok(await authenticationService.GetAccessToken());  
});  
  
app.MapGet("/signin-oidc", async (string code, string state, AuthenticationService authenticationService) =>  
{  
    await authenticationService.FetchAcessTokenFromCode(code, state);  
    return Results.Redirect("/");  
}).WithDisplayName("Redirect");
```

Besides the performance improvements, one of the most liked features of .NET 6 is reduced boilerplate code needed for an application.

For keeping the configuration here, and in the Worker Service, I follow the convention of `appsettings.json` and extracting config from it like this:
```csharp
AuthenticationConfiguration? config = new AuthenticationConfiguration  
(  
    builder.Configuration["Authentication:TenantId"],  
    builder.Configuration["Authentication:ClientId"],  
    builder.Configuration["Authentication:ClientSecret"],
);  
  
builder.Services.AddSingleton(config);
```

In the Worker Service, instead of pulling field by field I get an entire section from `appsettings.json`:
```csharp
var cronConfiguration = configuration.GetSection("Cron").Get<CronConfiguration>();
```

In the endpoints used for the OAuth 2.0 flow, access-token and signin-oidc, I make calls to an AuthenticationService instance that is injected into the HTTP Action methods mapping. The flow is pretty standard, after the initial redirection to the Auth Server (`login.microsoftonline.com/consumers`) the browser agent gets redirected back to the Client application which takes the auth code and makes a request to the Auth Server for an Access Token. 
This token is then kept in a singleton and returned in the access-token endpoint, unless it is expired. The the Client application just requests for a new pair of tokens--access and refresh.
Refresh tokens can be exchanged for 90 days, from the day that the session was initially established.

Before implementing all of this I had to make an App Registration on Azure AD, for Personal Microsoft accounts. I used a localhost redirect URI through HTTP. This was in line with my threat model as this is going to be locally hosted on an Raspberry Pi and accessible on that Pi. During development I found it quite easy to authenticate myself by simply SSH-ing into the RPi through VS Code and then port forwarding the session gateway app.

To access files and refresh the session I had to add the following permissions:
- Files.Read
- Files.Read.All
- offline_access
- openid
- User.Read

# Using the Graph API SDK
The official SDK simplifies interaction with the Graph API--my main reason for using it over using a HttpClient instance. To authenticate yourself, or your app, with it there are a number of auth providers that you can use. However, as my case is specific in that the Worker Service requests the access token from the session gateway application, I had to customize Graph SDK service client. 

```csharp
_httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer",  
    await _sessionService.GetAccessTokenAsync(stoppingToken));  
  
// This fixes a GraphServiceClient that is caused by the fact that this custom auth implementation has a null auth provider.  
_httpClient.DefaultRequestHeaders.Add(CoreConstants.Headers.FeatureFlag,  
    Enum.Format(typeof(FeatureFlag), FeatureFlag.AuthHandler, "x"));
    
_graphServiceClient = new GraphServiceClient(_httpClient);
```

Inside of of the GraphService class a private method instantiates a new GraphServiceClient with a HttpClient instance that has the access token. The line of code between them fixes an issue with this approach, where the GraphServiceClient will throw an exception due to a missing auth provider.

The HttpClient is instantiated with HttpClientFactory pattern:
```
services.AddHttpClient<GraphService>().SetHandlerLifetime(Timeout.InfiniteTimeSpan)  
    .ConfigureHttpClient(x => x.Timeout = TimeSpan.FromMinutes(3));
```

After noticing that some requests towards the Graph API tended to hand for a minute or so I added the increased timeout.

## Fetching the file list
The first part of the download process, for OneDrive, is to get a list of all files including their name, size and path. The SDK doesn't have a method of doing this in one step, but it can be done by simply recursing through the directories--starting at root.

```csharp
string? folderPageId = (await _graphServiceClient.Me.Drive.Root  
    .Request()  
    .GetAsync()).Id;  
  
  
List<FileDownload>? files = new List<FileDownload>();  
  
files.AddRange(await GetFilesFromFolderPageRecusivelly(folderPageId, "", stoppingToken));
```

Initially I fetch the Id of the root and then through the recursive method populate a list of the FileDownload model (which keeps the file name, size, and path properties). The recursive method uses the `_graphServiceClient.Me.Drive.Items[folderId].Children` property to get all items inside of a directory, adds to list all non-directory items and calls the recursive method adding the directory name as the string parameter. This is done to construct the path as this property is not part of the Children request response and  making a new request for every directory item just for the path would take too much network resources.

## Downloading the files
Finally, from a loop in the OneDriveService I call the download method, supplying it with the item id.

This methods simply returns stream of an item:
```csharp
return await _graphServiceClient  
    .Me.Drive.Items[fileId].Content  
    .Request()  
    .GetAsync();
```

Not shown here but the method also has a try catch block that catches a ServiceException that might get triggered if the token is expired and re-auths the client and attempts the download again.

# Next Steps
I'll try to wrap up the series in the next post where I'll go through logging, exception handling, network resiliency, testing, and a few other stuff.