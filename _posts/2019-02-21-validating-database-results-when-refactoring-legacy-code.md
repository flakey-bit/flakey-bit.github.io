---
id: 132
title: Validating database results when refactoring legacy code
date: 2019-02-21T15:06:13+10:00
author: eddiewould
layout: post
guid: http://feedingthe.code.blog/?p=52
permalink: /2019/02/21/validating-database-results-when-refactoring-legacy-code/
timeline_notification:
  - "1550725576"
categories:
  - Uncategorized
tags:
  - hashing
  - refactoring
  - sql
  - testing
---
Data-hashing techiques that can be used to validate database results in the absense of automated (unit) tests when refactoring.

Recently I was working on a legacy application in the financial services sector - the code in question performs a hypothetical ("what if") valuation of gas contracts over a period of time. Not really a mark-to-market but rather a set of calculated values (quantity allocated, liquidated charges etc).

There was nothing particularly complicated about the calculation (no multiple scenarios, no Monte-Carlo or anything like that) yet it was taking _hours_ (*10* hours for 30 contracts over 6 months). That's truely shocking by anyone's standards.

Clearly, the code needed refactoring. Whenever I'm faced with refactoring complex functionality the first thing I reach for is tests that show clear inputs and expected outputs. In this case I'd liked to have seen unit tests covering the internals of the calculation *and* integration tests covering the entire calculation run (given some contracts, a specific valuation environment, we expect _these_ results for a run of _n_ days).

Unfortunately, the code had no unit tests and did not lend itself to unit testing (it was _liberally sprinkled_ with random database reads and writes, essentially using the database as working memory).

<img src="/wp-content/uploads/2019/02/salt-bae-001.jpg" alt="Salt-Bae-001" width="800" height="450" class="alignnone size-full wp-image-57" />

Yet the business depended on the results being in the database. And the existing code (whilst slow) was producing correct results.

The question was, *how could I perform the minimum refactoring to speed the runs up whilst being confident I hadn't broken things*?

The first step was identifying the tables that the process was writing to. There was a table housing the _configuration_ of the valuation (with a unique identifier), however this table was not written to when _running_ the calculation.

The actual results were written to 8 tables (which I discovered from reading the code and following foreign key constraints). Fortunately, running a calculation only resulted in _inserts_ into these 8 tables (no updates). However if results already existed (from a previous attempt) then the run would fail. My first change was to modify the code to delete the relevant valuation run results (if present) before commencing.

I worked through each of the 8x tables and determined which columns were material/significant. Basically that meant everything except things like run timestamps and primary keys of the result tables.

#### GasValuationDealRunningTotals:

<img class="alignnone size-full wp-image-53" src="/wp-content/uploads/2019/02/gasvaluationdealrunningtotals.png" alt="GasValuationDealRunningTotals" width="532" height="48" />

#### GasValuationCalculationRuns:

<img src="/wp-content/uploads/2019/02/gasvaluationcalculationruns-1.png" alt="GasValuationCalculationRuns" width="1184" height="78" class="alignnone size-full wp-image-59" />

I then wrote some checksum queries as follows:

```sql
SELECT CHECKSUM_AGG(RowHash) AS DealRunningTotalCheck FROM (  
    SELECT CONVERT(INT, hashbytes('MD5',(
        SELECT
            drt.Deal_Id,
            [Type],
            SwapImbalanceQuantity,
            GasStorageQuantity        
        FROM (values(null))foo(bar) for xml auto
    ))) AS RowHash
    FROM Contract.GasValuationDealRunningTotals as drt
    JOIN Contract.GasValuationCalculationRuns cr ON cr.Id = drt.CalculationRun_Id 
    WHERE GasContractValuationId = @gasContractValuationId
) subquery
```

```sql
SELECT CHECKSUM_AGG(RowHash) AS CalculationRunCheck FROM (  
    SELECT CONVERT(INT, hashbytes('MD5',(
        SELECT
             Deal_Id,
             CompletionStatus,
             IsEndOfInvoicingPeriod,
             Error,
             AccountingDay,
             EstimatedInvoicePaymentDay    
        FROM (values(null))foo(bar) for xml auto
    ))) AS RowHash
    FROM Contract.GasValuationCalculationRuns cr
    WHERE GasContractValuationId = @gasContractValuationId
) subquery
```

Running these queries would spit out a single number which would give me relatively high confidence that the data matches:

<img src="/wp-content/uploads/2019/02/examplechecksums.png" alt="ExampleChecksums" width="172" height="106" class="alignnone size-full wp-image-56" />

For a given table, they would tell me

* If I was missing rows or had extra rows
* If the data in the rows did not match
* If I was failing to populate columns or populating columns when I should not be

whilst ignoring row order.

There are of course alternative approaches I could have taken, however given that I had 80,000 rows for a small dataset (only 3 months) keeping it within SQL server was preferable.

I then restored a production database backup and picked a calculation run that was representitive of the majority of runs & had run to completion successfullly. That calculation run (configuration) became my "baseline".

To give myself confidence in the approach I

* Ran the checksum queries against the existing calculation results (from production) & captured the checksums
* Deleted the relevant results rows from the database, ran the queries & verified the checkums all turned to NULL
* Ran the calculation run again (without any code modifications) and executed the checksum queries again - the checksums matched those from before *WOOHOO!*

As well as my "golden" (baseline) configuration, I also created some "mini" configurations (only a couple of contracts; valuation period of only a few days rather than several months) and captured the hashes for those before making changes.

I was then free to start refactoring/modifying the code with relative certainty that I wasn't breaking anything.

HACK HACK HACK...(many hours later)...

I managed to get the run time down from the initial 10 hours to *38 minutes* (will talk about how in a separate post). And low and behold, the checksum queries still match!

In the absense of any automated tests I of course had to get the business to manually verify things once I was done.

NOTES:
My initial approach used the SQL Server <a href="https://docs.microsoft.com/en-us/sql/t-sql/functions/checksum-transact-sql?view=sql-server-2017">CHECKSUM</a> function however I learned that CHECKSUM (whilst more convenient) has a high frequency of collisions (compared with HASHBYTES).

The trick for converting the row to XML before passing to HASHBYTES came from this page: <a href="https://www.sqlservercentral.com/Forums/1725965/Suggestionsolution-for-using-Hashbytes-across-entire-table" target="_blank" rel="noopener noreferrer">https://www.sqlservercentral.com/Forums/1725965/Suggestionsolution-for-using-Hashbytes-across-entire-table</a>

I decided that using CHECKSUM_AGG (over the HASHBYTES results) was probably safe-enough for my purposes.