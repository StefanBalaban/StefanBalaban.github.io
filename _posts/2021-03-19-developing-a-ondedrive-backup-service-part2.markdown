---
layout: post
title:  "Developing a OneDrive Backup Service With .NET 6: Part 2 Memory Paging"
date:   2022-03-19 16:22:12 +0100
categories: C# .NET 
---
Last post I went over the idea and basic setup behind this project. In this post I'll go about the specifics of the memory paging I implemented.

Memory paging usually refers to storing main memory in the secondary storage, here I use it to refer of storing the downloaded data to disk during the backup process.

During design I planned to download a batch of files, for each service, and then write that batch to a temp location, repeating until there are no more files for that service. Each batch would be of a size that is less than the maximum page size which would be less than the device's memory. Preventing an OutOfMemoryException.

```csharp
while (!service.AllFilesDownloaded)
{	
	 var files = await service.DownloadFilesAsync(stoppingToken);
	 for (int i = 0; i < files.Count; i++)
	
	 {
	 await fileStorageService.WriteFileToDirectoryAsync(files[i], directory.TempPath); 
	 } 
	
	 // Remove reference from the current list of downloaded files to promote it to Garbage
	 files = new List<FileDownload>();
	
	 // Force GC to make sure the memory is cleared of garbage.
	 // The performance penalty hit is acceptable to prevent memory being cluttered with Garbage
	 GC.Collect();
	 GC.WaitForPendingFinalizers();
}
```
This approach works but would have issues if there are files larger than the set page size. However, as I do not have any files over 500 MB and most of the services fetch a larger number of smaller size file, I left it as it is.

An obvious solution to this issue would be use the standard way of writing a stream that is being downloaded to a temp file as it is being downloaded.

During test runs I noticed that even after setting files to a new instance the Garbage Collector did not clear out objects holding the file streams, so I decided to force the collection.

# OneDrive
The classes that implement the file download service internally take care of downloading the files from the cloud location.

For OneDrive I first fetch the Ids and size of all the files inside of the storage and then proceed to download them while they are within the page size.

When the size of the currently downloaded files is about to reach the page size the files are returned.

To track the current position of the file being downloaded I keep an index field. When this index is equal to the size of the list that track the Ids of files to download the AllFilesDownloaded is set to true, and the backup process is finished for the OneDrive.

# Next Steps
Interaction with the OneDrive is done through the Graph API SDK, using the GraphServiceClient class. In the [next post](http://stefanbalaban.com/c%23/.net/2022/04/07/developing-a-ondedrive-backup-service-part3.html) I'll go over my usage of this class and the customized HttpClient that I use for authentication.