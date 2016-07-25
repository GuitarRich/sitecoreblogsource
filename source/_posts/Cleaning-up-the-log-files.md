title: "Cleaning up the log files"
date: 2015-07-24 14:47:18
tags:
- Sitecore
- Log4Net
category: Sitecore
---

Since upgrading to Sitecore 8 one thing that I have noticed is that the log files can be quite noisey. In particular when the default log file is set to _INFO_ this line is written a lot... and I mean a LOT! 

```
Heartbeat 13:48:33 INFO  Trace: [Finally] System.Data.SqlClient.SqlCommand.ExecuteReader::Finally;
```

Here is a small example of a clean Sitecore installation, no extra code written at this time:

{% asset_img Too-many-log-entries.jpg "Too Many Log Entries" %}

As you can see, this is being written to the logs around 6 times every 2 seconds! 

After a bit of digging, I finally located the offending code. And here it is:

``` csharp
  /// <summary>
  /// Writes a string entry into the Sitecore log.
  /// 
  /// </summary>
  /// <param name="entry">Entry to append to the log.</param>
  private void InternalWriteEntry(string entry)
  {
    if (LoggerFactory.SitecoreTraceListener.recursiveCall)
      return;
    LoggerFactory.SitecoreTraceListener.recursiveCall = true;
    try
    {
      Log.Info("Trace: " + entry, (object) this);
    }
    finally
    {
      LoggerFactory.SitecoreTraceListener.recursiveCall = false;
    }
  }
```

The code is in **Sitecore.Diagnostics.LoggerFactory.SitecoreTraceListener**. It looks like it should be doing a **Log.Trace** rather than **Log.Info**, then it would obey the log level settings.

###Log4Net to the Rescue
Fortunately Sitecore uses log4net so we can do something about it. Adding the following config to the log4net configuration section stopped Sitecore from filling up the log files with this pointless line!

``` xml
	<logger name="Sitecore.Diagnostics.LoggerFactory.SitecoreTraceListener" additivity="false">
		<level value="NONE"/>
		<appender-ref ref="LogFileAppender" />
	</logger>
```

So if you are seeing this a lot in your log files, I hope this little tip helps!

-- Richard
