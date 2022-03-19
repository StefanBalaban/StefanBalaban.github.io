---
layout: post
title:  "Developing a OneDrive Backup Service With .NET 6: Part 1 Intro"
date:   2022-03-12 9:25:14 +0100
categories: C# .NET 
---
Recently I started working on building my home network, which includes a workstation PC, a Raspberry Pi as a NAS and a server for some self-hosted services I'm using.  One of these services, that I need, is a service that would periodically backup my cloud resources, like databases, git repo's, VM's, and general-purpose cloud storage like OneDrive on the NAS. 

I couldn't find a solution that did this out of the box and I had wish to try out .NET 6 so I decided to write my own.

Here I'll recount the steps I took during the whole process, my thinking, the obstacles I faced and how I approached them.
This will be a series of posts that will cover all the parts of the service and reasoning while developing them for the purpose of telling a story of how an application gets built from idea to deployment and maintenance.

First post will be about my setup, motivation, and creating the foundation for code.

You can find the source code on [GitHub](https://github.com/StefanBalaban/BackMeUpOneDrive).

# Setup
I usually use Visual Studio Code on my laptop with my workstation PC as a remote host. However, I decided to use Visual Studio 2022 IDE for this undertaking as I would need more firepower in terms of IDE support with memory profiling and IntelliSense suggestions. Plus, I wanted to see in action the AI support and how helpful it is. 
As for .NET, version 6.0.2 recently came out so that is the one I used.
My only real constraint was time, I have about 2 hours that I can work after I'm done with my job. On Saturday I'd spend some more time to dig deep into blockers that accumulated during the work week.
To keep a track of my work, I used Obsidian by creating a note where I kept all things related to this effort and would chronologically add entries for each day, I did something. This way I could I stop and start without losing track.

# Initial Design

In my head I had a very abstract view of what this service would do: periodically pulling the data from cloud services, zipping them and storing them on the NAS. 

Before writing any code, I decided to first create a simple plan and design so my efforts would be organized. This would also help me prevent any scope creep by just developing the essentials. I didn't try to sketch out the whole service, instead I just created a high-level design of components I needed to accomplish my idea. The details would come in later when I started working on the actual component.

Using pen and paper I sketched out a flow that consisted of a main service that calls sub-services that are responsible for fetching data from a specific cloud service. For example, a OneDrive service, Azure service, Postgres service. The main service would go on, taking this data, zipping it and then sending it to the NAS.

I decided to use the Worker Service .NET template for this, as it would be a background task working without any user interaction. For deployment I would use Docker, containerizing the application.

# Development

With this in mind I started with the actual development. With `dotnet new` I created the project and then with Visual Studio imported it into a solution.
A new Worker Service project only has the Program.cs, Worker.cs, and appsettings.json files. Program.cs instantiates a Host with the Worker class as a HostedService and runs it.
The Worker itself has a method, ExecuteAsync, that is called when the program starts. This is where all the action happens.
First change was to add scheduled runs. I follow [this guide](https://medium.com/@gtaposh/net-core-3-1-cron-jobs-background-service-e3026047b26d) to create a simple cron based schedule that would run inside of the ExecuteAsync method loop. 
After adding this I started writing the high-level flow, using two interfaces which were placeholders at this time (one for file downloading the other for file storage). With this approach I could lay the groundwork for the classes that would implement this behavior later. This is, in essence, the dependency inversion principle. 

For the download interface I ended up with a single method that would return a list of files (each contains name, size, byte stream and path within OneDrive). It also exposes a boolean that would be set to true once there are no more files to download. The reason for using this boolean is because I decided to implement my own custom memory paging--the service downloads a set number of files and then writes them to temp storage on disk. This way the download method can be while-looped until all files are downloaded and the risk of hitting an out-of-memory exception is avoided.

The storage interface exposes methods for creating the temp files, with the correct path to preserve directory structure, zips the files, sends them to the NAS (in this case using SMB), and finally removes the temp files and redundant backups.

Another purpose for using interfaces is that with this generalized approach I could inject multiple download services (OneDrive, PostgreSQL, repo's, etc.) and have them follow the same flow. 

## Dependency Injection
The worker instance is alive during the whole lifetime of the program, meaning it is a singleton. However, the services that do the work should only be scoped to a single scheduled run. To resolve this, I use the service locator pattern to create a scope and then instantiate the service classes:
```csharp
            using IServiceScope scope = _serviceProvider.CreateScope();
            var fileStorageService = scope.ServiceProvider.GetService<IFileStorageService>();
            var fileDownloadServices = scope.ServiceProvider.GetServices<IFileDownloadService>();
```

This happens inside of the method that gets called once a scheduled run starts.

And for service registration I used the default .NET DI container. The registration happens inside of the host configuration.
```csharp
IHost host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostContext, services) =>
    {

        services.AddHostedService<Worker>();
        services.AddScoped<IFileDownloadService, OneDriveService>();        
        services.AddScoped<IFileStorageService, FileStorageService>();
    })

```

And with this set the development of underlying logic and communication could begin

# Next Steps
In the [following post](http://stefanbalaban.com/c%23/.net/2022/03/19/developing-a-ondedrive-backup-service-part2.html) I'll go through the custom memory paging.