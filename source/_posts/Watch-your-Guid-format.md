title: "Watch your Guid format!"
date: 2016-06-02 14:22:48
tags:
category: Sitecore
---

Maybe this should be in a "how to break Sitecore" section!

**Question:** Which of these `Guid` formats is valid?

```csharp
var ex1 = new Guid("00000000000000000000000000000000");
var ex2 = new Guid("00000000-0000-0000-0000-000000000000");
var ex3 = new Guid("{00000000-0000-0000-0000-000000000000}");
var ex4 = new Guid("(00000000-0000-0000-0000-000000000000)");
var ex5 = new Guid("{0x00000000,0x0000,0x0000,{0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00}}");
var ex6 = new Guid("Why-Wont-You-Be-My-Friend?");
```

**Answer:** The first 5 are perfectly valid `Guid` formats, _unless_ you need to index those values with Sitecore!

#### ShortID.Encode is picky about formats!
When adding a `Guid` to an index Sitecore uses the `ShortID` data type instead of the full Guid, this `ShortID` formats the `Guid` without hyphens or braces. This is great for indexes, it makes the value easily searchable by both Lucene and SOLR. Its also pretty useful in other area's. You might use it to add your Item ID to a query string or link so that you don't have to Url encode the values etc...

So pretty useful!

BUT, the method to encode a `Guid` to a `ShortID` is not very forgiving. Here it is:  

```csharp
/// <summary>Converts a string to a ShortID.</summary>
/// <param name="guid">The GUID.</param>
/// <returns>The encode.</returns>
public static string Encode(string guid)
{
    Assert.ArgumentNotNull((object) guid, "guid");
    return guid.Substring(1, 8) + guid.Substring(10, 4) + guid.Substring(15, 4) + guid.Substring(20, 4) + guid.Substring(25, 12);
}
```

So you can see the problem!
* First it takes a string, not a Guid object. Yes there is an overload that takes a Guid and formats it as a string, but you are not forced to use that.
* Second, apart from checking if the value is null, no other validation is done. This code would pass all validation:
```csharp
var myShortId = ShortID.Encode("Will you be my friend?");
```
Right away, the code assumes that the string is the correct length and that all the parts of the `Guid` are in the correct place. *Any* deviation to that results in an untrapped exception. This means that in the above list, examples 1, 2 and 5 would all cause an exception in your code.

The big problem here is that if you programatically edit a link or list field, you **must** make sure that you format the `Guid` correctly, otherwise you will end up with some nasty errors in the crawling log file. Fortunately there is a helper method for this:

```
// Sitecore utility method
var guidString = MainUtil.GuidToString(myGuid);

// Or just to string it
var guidString2 = myGuid.ToString("B");
```

### TL/DR;
So if you are going to use `ShortID.Encode` - make sure you pass in the right format or you are going to get some unexpected results!