A library written in C# 3.0 that provides the following facilities:

  * Parsing of crontab expressions
  * Formatting of crontab expressions
  * Calculation of occurrences of time based on a crontab schedule

This library does not provide any scheduler or is not a scheduling facility like cron from Unix platforms. What it provides is parsing, formatting and an algorithm to produce occurrences of time based on a give schedule expressed in the crontab format:

```
* * * * *
- - - - -
| | | | |
| | | | +----- day of week (0 - 6) (Sunday=0)
| | | +------- month (1 - 12)
| | +--------- day of month (1 - 31)
| +----------- hour (0 - 23)
+------------- min (0 - 59)
```

Star (`*`) in the value field above means all legal values as in parentheses for that column. The value column can have a `*` or a list of elements separated by commas. An element is either a number in the ranges shown above or two numbers in the range separated by a hyphen (meaning an inclusive range). For more, see CrontabExpression.

Below is an example in [IronPython](http://en.wikipedia.org/wiki/IronPython) of how to use `CrontabSchedule` class from NCrontab to generate occurrences of the schedule `0 12 * */2 Mon` (meaning, _12:00 PM on Monday of every other month, starting with January_) throughout the year 2000:

```
IronPython 1.1 (1.1) on .NET 2.0.50727.1434
Copyright (c) Microsoft Corporation. All rights reserved.
>>> import clr
>>> clr.AddReferenceToFileAndPath(r'C:\NCrontab\bin\Release\NCrontab.dll')
>>> from System import DateTime
>>> from NCrontab import CrontabSchedule
>>> s = CrontabSchedule.Parse('0 12 * */2 Mon')
>>> start = DateTime(2000, 1, 1)
>>> end = start.AddYears(1)
>>> occurrences = s.GetNextOccurrences(start, end)
>>> print '\n'.join([t.ToString('ddd, dd MMM yyyy hh:mm') for t in occurrences])
Mon, 03 Jan 2000 12:00
Mon, 10 Jan 2000 12:00
Mon, 17 Jan 2000 12:00
Mon, 24 Jan 2000 12:00
Mon, 31 Jan 2000 12:00

```

Below is the same example in [F#](http://msdn.microsoft.com/en-us/fsharp/cc742182) Interactive (`fsi.exe`):

```
Microsoft (R) F# 2.0 Interactive build 4.0.40219.1
Copyright (c) Microsoft Corporation. All Rights Reserved.

For help type #help;;

> #r "NCrontab.dll"
-
- open NCrontab
- open System
-
- let schedule = CrontabSchedule.Parse("0 12 * */2 Mon")
- let startDate = DateTime(2000, 1, 1)
- let endDate = startDate.AddYears(1)
-
- let occurrences = schedule.GetNextOccurrences(startDate, endDate)
- occurrences |> Seq.map (fun t -> t.ToString("ddd, dd MMM yyy hh:mm"))
-             |> String.concat "\n"
-             |> printfn "%s";;

--> Referenced 'C:\NCrontab\bin\Release\NCrontab.dll'

Mon, 03 Jan 2000 12:00
Mon, 10 Jan 2000 12:00
Mon, 17 Jan 2000 12:00
Mon, 24 Jan 2000 12:00



val schedule : NCrontab.CrontabSchedule = 0 12 * 1,3,5,7,9,11 1
val startDate : System.DateTime = 1/1/2000 12:00:00 AM
val endDate : System.DateTime = 1/1/2001 12:00:00 AM
val occurrences : seq<System.DateTime>
```
A crontab expression are a very compact way to express a recurring schedule. A single expression is composed of 5 space-delimited fields:

```
MINUTES HOURS DAYS MONTHS DAYS-OF-WEEK
```

Each field is expressed as follows:

  * A single wildcard (`*`), which covers all values for the field. So a `*` in days means all days of a month (which varies with month and year).
  * A single value, e.g. `5`. Naturally, the set of values that are valid for each field varies.
  * A comma-delimited list of values, e.g. `1,2,3,4`. The list can be unordered as in `3,4,2,6,1`.
  * A range where the minimum and maximum are separated by a dash, e.g. `1-10`. You can also specify these in the wrong order and they will be fixed. So `10-5` will be treated as `5-10`.
  * An interval specification using a slash, e.g. `*/4`. This means every 4th value of the field. You can also use it in a range, as in `1-6/2`.
  * You can also mix all of the above, as in: `1-5,10,12,20-30/5`

The table below lists the valid values for each field:

| **Field**      |  **Range** | **Comment**                                                 |
|:---------------|:-----------|:------------------------------------------------------------|
| MINUTES        |    0-59    | -                                                           |
| HOURS          |    0-23    | -                                                           |
| DAYS           |    0-31    | -                                                           |
| MONTHS         |    1-12    | Zero (0) is not valid. Month names also accepted.           |
| DAYS-OF-WEEK   |    0-6     | Where zero (0) means Sunday. Names of days also accepted.   |

Two fields also accept named values in English: MONTHS and DAYS-OF-WEEKS. So you can use names like `January`, `February`, `March `and so on for MONTHS and `Monday`, `Tuesday`, `Wednesday` and so on for DAYS-OF-WEEK. The names are not case-sensitive and you can even use short forms like `Jan`, `Feb`, `Mar` or `Mon`, `Tue`, `Wed`. In fact, at the moment, the parser in NCrontab will use the first match that it finds. Consequently, if you specify just the letter `J` for the month, then it will be interpreted as January since it occurs before the months June and July. If you specify `Ju`, then June will be assumed for the same reason. However, you should stick to either the 3 letter abbreviations or the full name since that is the norm among cron implementations. Finally, you can also mix numerical and named values, as in `Jan,Feb,3,4,May,Jun,6`.
\ 



## Writing the Table-Valued Function in C# ##

The `SqlCrontab` class below implements a managed table-value function called `GetOccurrences` that relies NCrontab for its implementation. Given a crontab expression and a range of time, `GetOccurrences` will yield occurrences as a table with a single column typed as `DATETIME`:

```C#
namespace NCrontab.Samples
{
    #region Imports

    using System;
    using System.Collections;
    using System.Data.SqlTypes;
    using Microsoft.SqlServer.Server;
    using NCrontab;

    #endregion

    public static class SqlCrontab
    {
        [SqlFunction(FillRowMethodName = "FillOccurrenceRow", IsDeterministic = true, IsPrecise = true)]
        public static IEnumerable GetOccurrences(SqlString expression, SqlDateTime start, SqlDateTime end)
        {
            if (expression.IsNull || start.IsNull || end.IsNull)
                return new DateTime[0];

            try
            {
                var schedule = CrontabSchedule.Parse(expression.Value);
                return schedule.GetNextOccurrences(start.Value, end.Value);
            }
            catch (CrontabException)
            {
                return new DateTime[0];
            }
        }

        public static void FillOccurrenceRow(object obj, out SqlDateTime time)
        {
            time = (DateTime) obj;
        }
    }
}
```

For more information on how CLR-based table-value functions are implemented and work, see [CLR Table-Valued Functions](http://msdn.microsoft.com/en-us/library/ms131103.aspx) topic in [SQL Server Books Online](http://msdn.microsoft.com/en-us/library/bb545450.aspx).

Assuming you are in the same directory as where the above sample code resides in a C# source file, compile a library using the C# compiler as shown here:


```
csc /t:library /r:NCrontab.dll /out:NCrontab.Samples.dll *.cs
```

## Registering with SQL Server ##

Next, in a SQL Server database where you wish to use this function, execute the following SQL batch to register the assembly and function.

```SQL
CREATE ASSEMBLY [NCrontab.Samples] 
FROM 'NCrontab.Samples.dll' -- supply the path before the file name here
GO

CREATE FUNCTION CrontabSchedule(
    @Expression NVARCHAR(100), 
    @Start DATETIME, 
    @End DATETIME)
RETURNS TABLE (
    [Occurrence] DATETIME)
AS 
EXTERNAL NAME [NCrontab.Samples].[NCrontab.Samples.SqlCrontab].[GetOccurrences]
GO
```

SQL Server CLR integration is usually turned off by default so the above `CREATE ASSEMBLY` and `CREATE FUNCTION` will work but the function cannot be used until CLR integration is enabled. Unless it is already enabled, use the following batch to accomplish this:

```SQL
sp_configure 'clr enabled', 1;
GO
RECONFIGURE
GO
```

For more information, see [Enabling CLR Integration](http://msdn.microsoft.com/en-us/library/ms131048.aspx) topic in [SQL Server Books Online](http://msdn.microsoft.com/en-us/library/bb545450.aspx).

## Crontab Schedule Occurrences in SQL Queries ##

Now, the new `dbo.CrontabSchedule` can be used in a query. For example, the following query will yield all occurrences of February 29th between 1900 and 2000:

```SQL
SELECT * FROM dbo.CrontabSchedule('0 12 29 feb *', '1900-1-1', '2000-1-1')
GO
```

The result of the above query should look like this:

```
1904-02-29 12:00:00.000
1908-02-29 12:00:00.000
1912-02-29 12:00:00.000
1916-02-29 12:00:00.000
1920-02-29 12:00:00.000
...
1996-02-29 12:00:00.000
```

## Multiple Schedule Occurrences with Unions ##

Suppose you want to have a schedule that yields midnight on 1st, 2nd and 3d of February and then on 4th, 5th or 6th of March. Unfortunately, you cannot express this in a single crontab expression. What you can do, however, is use two expressions and `UNION` their results as shown in the example below :

```SQL
SELECT * FROM dbo.CrontabSchedule('0 12 1,2,3 feb *', '2000-1-1', '2005-1-1')
UNION
SELECT * FROM dbo.CrontabSchedule('0 12 4,5,6 mar *', '2000-1-1', '2005-1-1')
ORDER BY 1
GO
```

The result of the above query should look like this:

```
2000-02-01 12:00:00.000
2000-02-02 12:00:00.000
2000-02-03 12:00:00.000
2000-03-04 12:00:00.000
2000-03-05 12:00:00.000
2000-03-06 12:00:00.000
...
2001-03-05 12:00:00.000
```

## Conclusion ##

# Examples of crontab expressions #

Following are examples of crontab expressions and how they would interpreted as a recurring schedule.

```
* * * * *
```

This pattern causes a task to be launched every minute.

```
5 * * * *
```

This pattern causes a task to be launched once every hour and at the fifth minute of the hour (00:05, 01:05, 02:05 etc.).

```
* 12 * * Mon
```

This pattern causes a task to be launched every minute during the 12th hour of Monday.

```
* 12 16 * Mon
```

This pattern causes a task to be launched every minute during the 12th hour of Monday, 16th, but only if the day is the 16th of the month.

```
59 11 * * 1,2,3,4,5
```

This pattern causes a task to be launched at 11:59AM on Monday, Tuesday, Wednesday, Thursday and Friday. Every sub-pattern can contain two or more comma separated values.

```
59 11 * * 1-5
```

This pattern is equivalent to the previous one. Value ranges are admitted and defined using the minus character.

```
*/15 9-17 * * *
```

This pattern causes a task to be launched every 15 minutes between the 9th and 17th hour of the day (9:00, 9:15, 9:30, 9:45 and so on... note that the last execution will be at 17:45). The slash character can be used to identify periodic values, in the form of a/b. A sub-pattern with the slash character is satisfied when the value on the left divided by the one on the right gives an integer result (a % b == 0).

```
* 12 10-16/2 * *
```

This pattern causes a task to be launched every minute during the 12th hour of the day, but only if the day is the 10th, the 12th, the 14th or the16th of the month.

```
* 12 1-15,17,20-25 * *
```

This pattern causes a task to be launched every minute during the 12th hour of the day, but the day of the month must be between the 1st and the 15th, the 20th and the 25, or at least it must be the 17th.


---


The above examples were slightly adapted from [cron4j Quickstart](http://www.sauronsoftware.it/projects/cron4j/quickstart.php)
