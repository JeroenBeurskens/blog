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
I eventually figured out how it works. 
</p>
<p>
  The C# code below iterates trough pages of the recycle bin. This makes it possible to scan trough results and find what you are looking for without pulling in everything at once.
</p>
{% highlight ruby %}
{% raw %}
using (var currentContext = AppContext.CreateContext(Uri))
{  
      currentContext.Load(currentContext.Site.RootWeb, w => w.Url);
      currentContext.ExecuteQueryRetry();
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
                  nextPageInfo = $"id={deletedItems.Last().Id}&title={deletedItems.Last().Title}&searchValue={deletedItems.Last().DeletedDate.ToString("s")}";
                  lastId = deletedItems.Last().Id;
              }
              else
              {
                  break;
              }
          }
      }
  }
  {% endraw %}
  {% endhighlight %}
