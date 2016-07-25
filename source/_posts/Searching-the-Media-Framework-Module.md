title: "Searching the Media Framework Module"
date: 2015-11-19 16:23:54
tags:
- Sitecore
- Media Framework
- Ooyala
category: Sitecore
---

A number of our clients partner with video providers like Brightcove or Ooyala. To sync the content between Sitecore and the video provider, Sitecore have provided a module called the [**Media Framework**](https://sdn.sitecore.net/Products/Sitecore%20Media%20Framework.aspx). We are using the Ooyala Edition. The framework adds a new template, **OoyalaVideo**, and the content items are stores in an Item Bucket inside the Media Library.

{% asset_img MediaFrameworkTree.png "Media Framework Content Tree" %}

Because the items are in a bucket, to display a list we will use the `SearchAPI` to get lists of video items for display. The examples here are using **[Fortis](http://fortis.ws)** and **Lucene**, but can easily be modified to Glass/Synthesis or straight Sitecore, and adapted to a SOLR implementation. Here is the code for a simple paged list of video items ordered by date descending or ascending:

```csharp
public IEnumerable<IOoyalaVideo> GetVideos(
	int page,
	int pageSize,
	string sortBy,
	out int totalResults)
{
	using (var searchContext = this.SearchManager.SearchContext)
	{

		var query = this.ItemSearchFactory.Search(
			searchContext.GetQueryable<IOoyalaVideo>()
				.ContainsAnd(item => item.LabelsListValue, labelIds)
			);

		if (sortBy == "" || sortBy == PagedListHelper.SortByVideoDescending)
		{
			query = query.OrderByDescending(x => x.CreatedAt);
		}
		else if (sortBy == PagedListHelper.SortByVideoAscending)
		{
			query = query.OrderBy(x => x.CreatedAt);
		}

		var results = query.GetResults();
		totalResults = results.TotalSearchResults;

		return results.Hits.Skip(pageSize * (page - 1)).Take(pageSize).Select(x => x.Document).ToList();
	}
}
```

All seemed to be working ok, except the sorting. It had no effect on the results being returned at all. On further investigation it turns out that the `MediaFramework` adds some custom fields to Sitecore.

{% asset_img MediaFrameworkFields.png "Media Framework - Custom Fields" %}

On checking the index configuration - these new field types were not being added to the index, so none of the filters or sorts based on those filters were working.

So we updated the index configuration to have the new field types:

```xml
<configuration xmlns:patch="http://www.sitecore.net/xmlconfig/">
	<sitecore>
		<contentSearch>
			<indexConfigurations>
				<defaultLuceneIndexConfiguration type="Sitecore.ContentSearch.LuceneProvider.LuceneIndexConfiguration, Sitecore.ContentSearch.LuceneProvider">
					<fieldMap type="Sitecore.ContentSearch.FieldMap, Sitecore.ContentSearch">
						<fieldTypes hint="raw:AddFieldByFieldTypeName">
							<fieldType fieldTypeName="readonlytext" storageType="YES" indexType="UN_TOKENIZED" vectorType="NO" boost="1f" type="System.String"   settingType="Sitecore.ContentSearch.LuceneProvider.LuceneSearchFieldConfiguration, Sitecore.ContentSearch.LuceneProvider" />
							<fieldType fieldTypeName="readonlycheckbox" storageType="YES" indexType="UN_TOKENIZED" vectorType="NO" boost="1f" type="System.Boolean"   settingType="Sitecore.ContentSearch.LuceneProvider.LuceneSearchFieldConfiguration, Sitecore.ContentSearch.LuceneProvider" />
						</fieldTypes>
					</fieldMap>
				</defaultLuceneIndexConfiguration>
			</indexConfigurations>
		</contentSearch>
	</sitecore>
</configuration>
```

Rebuild the indexes and magically the sorting starts working again! Amazing what you can do when the fields are added into the index properly :)

###TL;DR
When you are using the **Media Framework** module, Brightcode or Ooyala editions, make sure that any custom fields are included in the index configuration by `fieldType` if you want to use them in your search queries.

####Credits:  
[@owenniblock](https://twitter.com/owenniblock) of [http://www.kumquatcomputing.co.uk/](http://www.kumquatcomputing.co.uk/) for helping get this issue fixed.
