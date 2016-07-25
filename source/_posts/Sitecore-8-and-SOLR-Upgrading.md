title: "Sitecore 8 and SOLR Upgrading"
date: 2015-01-14 09:36:14
tags:
- Sitecore
- Sitecore 8
- SOLR
category: Sitecore
---

Just read a great post by [Jon Stoneman](http://www.sequence.co.uk/blog/authors/jon-stoneman/) at Sequence on how to get [Sitecore 8 working with Solr](http://www.sequence.co.uk/blog/sitecore-8-and-solr/). Their experiences are very similar to mine, but I came across some other “gotchas” that I wanted to share.

#### Old Config FIles

In Sitecore 8 the Solr configuration files have changed a lot. The file name has changed, so you will need to make sure you remove the v7.x Solr config files.

#### Naming The Core

By default, Sitecore 8 uses a different Solr core for each database and each module, historically at Lightmaker we have used a single Solr core for all databases. Following Sitecore best practice, all the settings were made using include files. This means during upgrades we can just add the new config files from Sitecore and we don’t have to re-apply all our changes.

This was the config we had for Solr:

``` xml
<configuration>
	<indexes hint="list:AddIndex">
		<index id="sitecore_master_index" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
			<param desc="core">NameOfCore</param>
		</index>
		<index id="sitecore_web_index" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
			<param desc="core">NameOfCore</param>
		</index>
		<index id="sitecore_core_index" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
			<param desc="core">NameOfCore</param>
		</index>
	</indexes>
</configuration>
```

The problem here was that Sitecore 8 has added an extra indexes for analytics, FXM, testing etc... Unfortunately rather than getting an error stating that the core could not be found, you just get

>##### _Connection error to search provider [Solr] : Unable to connect to [http://localhost:8983/solr]_

Which is not that helpful. If you get this error and the Solr instance is running and you can load the specified Url, then it is probably the Solr Core name that is wrong.

All we needed to do was to add in the Solr Core names for the new indexes and we were up and running.

``` xml
<configuration>
	<indexes hint="list:AddIndex">
        <index id="sitecore_master_index" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
            <param desc="core">NameOfCore</param>
        </index>
        <index id="sitecore_web_index" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
            <param desc="core">NameOfCore</param>
        </index>
        <index id="sitecore_core_index" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
            <param desc="core">NameOfCore</param>
        </index>
        <index id="sitecore_analytics_index" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
            <param desc="core">NameOfCore</param>
        </index>
        <index id="sitecore_testing_index" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
            <param desc="core">NameOfCore</param>
        </index>
        <index id="sitecore_suggested_test_index" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
            <param desc="core">NameOfCore</param>
        </index>
        <index id="sitecore_fxm_master_index" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
            <param desc="core">NameOfCore</param>
        </index>
        <index id="sitecore_fxm_web_index" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
            <param desc="core">NameOfCore</param>
        </index>
        <index id="sitecore_list_index" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
            <param desc="core">NameOfCore</param>
        </index>
        <index id="social_messages_master" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
            <param desc="core">NameOfCore</param>
        </index>
        <index id="social_messages_web" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
            <param desc="core">NameOfCore</param>
        </index>
        <index id="sitecore_marketing_asset_index_master" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
            <param desc="core">NameOfCore</param>
        </index>
        <index id="sitecore_marketing_asset_index_web" type="Sitecore.ContentSearch.SolrProvider.SolrSearchIndex, Sitecore.ContentSearch.SolrProvider">
            <param desc="core">NameOfCore</param>
        </index>
	</indexes>
</configuration>
```

– Richard Seal

