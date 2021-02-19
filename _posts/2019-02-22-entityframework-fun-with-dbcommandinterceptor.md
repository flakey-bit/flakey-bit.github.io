---
id: 75
title: 'EntityFramework &#8211; fun with DbCommandInterceptor'
date: 2019-02-22T09:43:00+10:00
author: eddiewould
layout: post
guid: http://feedingthe.code.blog/?p=75
permalink: /2019/02/22/entityframework-fun-with-dbcommandinterceptor/
timeline_notification:
  - "1550792584"
ocean_gallery_link_images:
  - 'off'
ocean_sidebar:
  - "0"
ocean_second_sidebar:
  - "0"
ocean_disable_margins:
  - enable
ocean_display_top_bar:
  - default
ocean_display_header:
  - default
ocean_center_header_left_menu:
  - "0"
ocean_custom_header_template:
  - "0"
ocean_header_custom_menu:
  - "0"
ocean_menu_typo_font_family:
  - "0"
ocean_disable_title:
  - default
ocean_disable_heading:
  - default
ocean_disable_breadcrumbs:
  - default
ocean_display_footer_widgets:
  - default
ocean_display_footer_bottom:
  - default
ocean_custom_footer_template:
  - "0"
ocean_link_format_target:
  - self
ocean_quote_format_link:
  - post
categories:
  - Uncategorized
tags:
  - DbCommandInterceptor intercept EntityFramework logging SQL exceptions parameters performance
---

(Ab)using `DbCommandInterceptor` in EntityFramework for fun and profit.

As a relative newcomer to EntityFramework (coming from a background of plain SQL) I often find myself frustrated by the "black box" experience it provides. Of course you're going to tell me to

<figure class="wp-block-image"><img src="/wp-content/uploads/2019/02/read-the-source.jpg" alt="read the source" class="wp-image-76"/></figure>

And I do. Sometimes I'll clone my own copy of the source & compile & link against that so that I can debug it in the context (no pun intended) of my application.

Recently I discovered a useful extensibility point - `DbInterception` which (among many other uses) can give insight into how EF is actually interacting with your database.

In a nutshell, the interceptor provides hooks for

* When EntityFramework is about to issue a query/command &
* After EntityFramework has executed a query/command

It provides access to the `DbCommand` - which includes the command text, the parameters etc) and a `DbCommandInterceptionContext` - which includes the query/command result, the exception thrown (if any), and other goodies.

The simplest usage is to register an interceptor in your application's entry point (`Program.cs`, `Startup.cs`, `Global.asax.cs` etc) and leave it there.

```csharp
public class Program
{
    public static void Main() {
        DbInterception.Add(new MyDBCommandInterceptor());
    }
}
```

The interceptor must implement `System.Data.Entity.Infrastructure.Interception.IDbCommandInterceptor` - the easiest way is to derive from `DbCommandInterceptor` (same namespace) which implements the interface as virtual methods.

### Richer exception logging

Have you ever seen an exception like `System.Data.SqlClient.SqlException: The INSERT statement conflicted with the FOREIGN KEY constraint "FK_SomeTable_SomeColumn". The conflict occurred in database "somedb", table "dbo.SomeTable", column 'Id'` in your logs?

If you have, I'm betting you wish you knew what value of `Id` it was trying to insert at the time.

By hooking into `NonQueryExecuted`/`ReaderExecuted`/`ScalarExecuted` you can log

* The stacktrace
* The command/query text that was being executed
* The *parameters and values* that were passed to the command/query

for commands/queries that fail under EF (this example uses Serilog & it's destructuring).


```csharp
public class MyDBCommandInterceptor : DbCommandInterceptor
{
  public override void NonQueryExecuted(DbCommand command, DbCommandInterceptionContext<int> interceptionContext)
  {
      base.NonQueryExecuted(command, interceptionContext);
      MaybeLogException(command, interceptionContext);
  }

  public override void ReaderExecuted(DbCommand command, DbCommandInterceptionContext<DbDataReader> interceptionContext)
  {
      base.ReaderExecuted(command, interceptionContext);
      MaybeLogException(command, interceptionContext);
  }

  public override void ScalarExecuted(DbCommand command, DbCommandInterceptionContext<object> interceptionContext)
  {
      base.ScalarExecuted(command, interceptionContext);
      MaybeLogException(command, interceptionContext);
  }

  private void MaybeLogException<TContext>(DbCommand command, DbCommandInterceptionContext<TContext> interceptionContext)
  {
      if (interceptionContext.Exception != null)
      {
          Log.ForContext("CommandText", command.CommandText).ForContext("Parameters", command.Parameters, true)
              .Error(interceptionContext.Exception, "Exception executing SQL command");
      }
  }
}
```

Note that
* There is no try/catch (instead, we look for the exception on the interception context)
* We need to override the ...Execut*ed* methods, rather than the ...Execut*ing* methods (because the exception won't be present before the command was executed!)

### Time consuming / frequent queries

Perhaps while running your application you've noticed SQL Server is very busy - you're wondering what the heck your application is doing to create all that load?

An interceptor such as the one below can reveal which queries are consuming those precious CPU minutes.

```csharp
public class MyDBCommandInterceptor: DbCommandInterceptor
{
	public static ConcurrentDictionary CommandStartTimes = new ConcurrentDictionary();
	public static ConcurrentDictionary CommandDurations = new ConcurrentDictionary();

	public override void NonQueryExecuting(DbCommand command, DbCommandInterceptionContext interceptionContext) {
		CommandStartTimes.TryAdd(command, DateTime.Now);
		base.NonQueryExecuting(command, interceptionContext);
	}

	public override void ReaderExecuting(DbCommand command, DbCommandInterceptionContext interceptionContext) {
		CommandStartTimes.TryAdd(command, DateTime.Now);
		base.ReaderExecuting(command, interceptionContext);
	}

	public override void ScalarExecuting(DbCommand command, DbCommandInterceptionContext interceptionContext) {
		CommandStartTimes.TryAdd(command, DateTime.Now);
		base.ScalarExecuting(command, interceptionContext);
	}

	public override void NonQueryExecuted(DbCommand command, DbCommandInterceptionContext interceptionContext) {
		base.NonQueryExecuted(command, interceptionContext);
		AccumulateTime(command);
	}

	public override void ReaderExecuted(DbCommand command, DbCommandInterceptionContext interceptionContext) {
		base.ReaderExecuted(command, interceptionContext);
		AccumulateTime(command);
	}

	public override void ScalarExecuted(DbCommand command, DbCommandInterceptionContext interceptionContext) {
		base.ScalarExecuted(command, interceptionContext);
		AccumulateTime(command);
	}

	private void AccumulateTime(DbCommand command) {
		if (CommandStartTimes.TryRemove(command, out
		var commandStartTime)) {
			var commandDuration = DateTime.Now - commandStartTime;
			CommandDurations.AddOrUpdate(command.CommandText, commandDuration, (_, accumulated) => commandDuration + accumulated);
		}
	}
}
```

The `CommandDurations` dictionary will contain a running map of how much time has been spent in each query/command (keyed by the CommandText).

```csharp
public class Program {
	public static void Main() {
		DbInterception.Add(new MyDBCommandInterceptor());
		// Do some heavy work that makes a bunch of EF queries / executes commands...
		// ...
		// pair.Key is the query/command text; pair.Value is the accumulated time spent executing that query/command
		foreach(var pair in MyDBCommandInterceptor.CommandDurations.OrderByDescending(pair => pair.Value).Take(5)) {
			Log.ForContext("Duration", pair.Value).ForContext("Query", pair.Key).Information("Expensive query");
		}
	}
}
```

### "What the heck is making this query"

Following on from "Time consuming / frequent queries" above, perhaps you've identified a query that is causing a performance problem but you're not sure where in the code the query is coming from (and you're too lazy to read through the code to figure it out)?

No problem, just set a conditional breakpoint in the interceptor. You could set the breakpoint condition on either

* Accumulated time being over some limit (e.g. 10 minutes)
* The command text containing some particular identifying substring

When the breakpoint trips you can examine the stack and see where it's coming from.

### Other uses

I'm sure I've just scratched the surface of possible uses of this extensibility point - for example it's possible to do things like supressing the execution of the command and set the result manually. You could possibly implement some kind of caching or mocking with this:

```csharp
public override void ReaderExecuting(DbCommand command, DbCommandInterceptionContext interceptionContext) {
	var dataTableResult = new DataTable("MyTable");
	interceptionContext.SuppressExecution();
	interceptionContext.Result = new DataTableReader(dataTableResult);
	base.ReaderExecuted(command, interceptionContext);
}
```


Keen to hear about other uses in the comments!