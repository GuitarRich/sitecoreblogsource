title: "Sitecore - Sorting by Date with SOLR"
date: 2014-11-06 09:35:25
tags:
- Sitecore 8
- Item Buckets
- SOLR
category: Sitecore
---

With the introduction of Item Buckets and the fantastic Sitecore Search API in Sitecore 7, we are using the search API more and more to get lists of content. Recently I was working on a site that had a massive news article repository (000’s of articles). These made a great candidate for Item Buckets – and all was working well until we wanted to pull out the top 10 most recent articles. Simple you might think, and the code was pretty simple:

#### Search result poco:

``` csharp
public class NewsArticleSearchResultItem : SearchResultItem
{
	[IndexField("authored_date")]
	public DateTime AuthoredDate { get; set; }
}
```
#### Search code:
``` chsarp
using (var context = ContentSearchManager.GetIndex("sitecore_web_index").CreateSearchContext())
{
    var searchQuery = context.GetQueryable<NewsArticleSearchResultItem>()
        .Where(x => x.TemplateID = NewsArticleTemplateId)
        .OrderByDescending(x => x.AuthoredDate)
        .Take(10)
        .ToList();
}
```

#### Results:
* Article 1: Date 2014/11/01
* Article 2: Date 2014/11/02
* Article 3: Date 2014/11/03

But the results are not sorted. I tried a few different ways, ascending and descending sorts, but the list was always in the same order.

### Stackoverflow to the Rescue

I found this post: [Solr sort does not seem to work](http://stackoverflow.com/questions/14035043/solr-sort-does-not-seem-to-work) on stack overflow which lead me to the fix.  
 According to the Solr [documentation](http://wiki.apache.org/solr/CommonQueryParameters#sort) –  
 _“Sorting can be done on the “score” of the document, or on any multiValued=”false” indexed=”true” field provided that the field is either non-tokenized (ie: has no Analyzer) or uses an Analyzer that only produces a single Term (ie: uses the KeywordTokenizer).”_

### The Fix

To fix this we need to add the sort field to the Solr Schema.xml and copy the data from the original field.
``` xml
<field name="authored_date_tdt_sort" type="tdate" indexed="true" stored="false" />
<copyField source="authored_date_tdt" dest="authored_date_tdt_sort" />
```
Once you have added the fields to the Schema.xml, reload your Solr core and rebuild your indexes. Now you can sort by the new field.

#### Search result poco:

``` csharp
public class NewsArticleSearchResultItem : SearchResultItem
{
	[IndexField("authored_date")]
	public DateTime AuthoredDate { get; set; }

	[IndexField("authored_date_tdt_sort")]
	public DateTime AuthoredDateSort { get; set; }
}
```

#### Search code:

``` csharp
using (var context = ContentSearchManager.GetIndex("sitecore_web_index").CreateSearchContext())
{
    var searchQuery = context.GetQueryable<NewsArticleSearchResultItem>()
        .Where(x => x.TemplateID = NewsArticleTemplateId)
        .OrderByDescending(x => x.AuthoredDateSort)
        .Take(10)
        .ToList();
}
```

#### Results:
* Article 3: Date 2014/11/03
* Article 2: Date 2014/11/02
* Article 1: Date 2014/11/01

Perfect – works exactly how we wanted it to. The copy field, copies the data from the dynamic field added to the index by the Sitecore crawler and adds it to the non-stored custom field. We have to add the field to the search result poco class separately as the field is not stored and so the property cannot be used to display the content. The field is only to be used for sorting.

Until next time…

– Richard Seal