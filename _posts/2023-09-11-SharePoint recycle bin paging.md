---
title: "SharePoint recycle bin paging"
date: 2023-09-11
render_with_liquid: false
---
SharePoint recycle bin paging
---
<p>
I was trying to create a backend function to scan sites for specific items. It seems there is no real way to query a recycle bin without loading everything in and looping trough the Recyclebin items.
While this is enough in most scenarios it can cause a problem with recyclebins with many items. A partial solution to this is to use paging. The RecycleBinQueryInformation class does have a PagingInfo property, but there is no documentation on it.
I eventually figured out how it works. Tested on on-prem 2019, but should work for other versions as well.
</p>
<p>
  The C# code below uses CSOM to iterate trough pages of the recycle bin. This makes it possible to scan trough results and find what you are looking for without pulling in everything at once.
</p>
{% highlight ruby %}
{% raw %}
      using (var siteContext = new ClientContext("https://YourSite"))      
      {
          Guid lastId = Guid.Empty;
          int recycleBinQueryPageSize = 50;
          string nextPageInfo = string.Empty;  
          List<RecycleBinItem> results = new List<RecycleBinItem>();  
            
          while (true)
          {
              // create the query
              var recycleBinQuery = new RecycleBinQueryInformation();
              recycleBinQuery.ShowOnlyMyItems = false;
              recycleBinQuery.ItemState = RecycleBinItemState.FirstStageRecycleBin;
              recycleBinQuery.OrderBy = RecycleBinOrderBy.DeletedDate;
              recycleBinQuery.IsAscending = false;
              recycleBinQuery.RowLimit = recycleBinQueryPageSize;

              // PagingInfo is where the magic happens. Completely undocumented unfortunately.
              // Pass an empty tring for the first page. After we receive the first page the PagingInfo for the next page will be derived from the results.
              recycleBinQuery.PagingInfo = nextPageInfo;
              
              RecycleBinItemCollection deletedItems = siteContext.Web.GetRecycleBinItemsByQueryInfo(recycleBinQuery);  
              
              siteContext.Load(deletedItems,
                  i => i.Include(
                      j => j.Id,
                      j => j.Title,
                      j => j.Author.LoginName,
                      j => j.DeletedDate,
                       j => j.DeletedBy.LoginName,
                      j => j.ItemType
                      ));
                      
              siteContext.ExecuteQueryRetry();  
              
              // Process the results. Filter out the last id. The the newest page will also contain the last entry of the previous page.
              results.AddRange(deletedItems.Where(i => i.Id != lastId));  
              
              // Found a full page. Prepare paginginfo for the next page
              if (deletedItems.Count >= recycleBinQueryPageSize)
              {
              // pageinfo format. Contains data from the last item on the page that is currently loaded.
              // This will be used as a starting point for the next page.
              // A similar url can be seen in the browser (F12) network traffic when browsing a large recyclebin using the SharePoint UI.
                  nextPageInfo = $"id={deletedItems.Last().Id}&title={deletedItems.Last().Title}&searchValue={deletedItems.Last().DeletedDate.ToString("s")}";
                  lastId = deletedItems.Last().Id;
              }
              else
              {
                  break;
              }
          }
      }
  {% endraw %}
  {% endhighlight %}
