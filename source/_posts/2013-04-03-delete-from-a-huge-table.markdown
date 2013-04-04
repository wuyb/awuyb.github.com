---
layout: post
title: "delete from a huge table"
date: 2013-04-03 01:31
comments: true
categories: [database]
---

Let's say I am hired to manage all trees in a forest. One of my job is to monitor the height of the trees at different time. The height is measured by a mysterious sensor which sends the data in with a non-deterministic pattern. The data is stored in a table called **HeightOfTrees**. Now and then, trees are cut down. I was just too lazy to remove them from the system. But today, the boss came and found out the inconsistency. He shouted and asked me to clean up the database.

The structure of **HeightOfTrees** is something like this:
``` sql
CREATE TABLE HeightOfTrees (
    tree_id   INT,
    timestamp TIMESTAMP,  -- timestamp is truncated to second
    height    FLOAT       -- height is truncated to centimeter
);
```
Yes, that's it! We have no primary key, no index, no nothing!  Never mind, at first glance, it looks like pretty straightforward: `DELETE FROM HeightOfTrees WHERE tree_id IN (SELECT tree_id FROM tree WHERE status='DOWN');` Okay, let's run it! Tick, tick, tick... after about two hours, suddenly, the whole system depending on the database is down! 'DB error' is everywhere. I realize this has something to do with my delete statement, which is still running, but, in the DB log, it shows something like 'rolling back transactions'. Why? From the deepest  place of my brain, I realize this might has something to do with how transactions are implemented.

The database management system (DBMS) that we are using is transactional, and its transaction is implemented by applying the famous [Write-Ahead Logging](http://en.wikipedia.org/wiki/Write_ahead_logging) paradigm. So, before the transaction is committed, the log which contains the information in the form of `(Sequence Number, Transaction ID, Page ID, Redo, Undo, Previous Sequence Number)` should be stored. Simple math tells me that the log file will be at least twice as large as the data to be deleted itself! As the log space is limited, and I have billions of records to be deleted from that table, DBMS is not able to persist the logs, so it simply cancels the transaction and rolls it back. Now what? Patient, as we have to let the DBMS do its job to make sure the state is **consistent**.

While waiting for the rollback to finish, a new idea pops out of mind and quickly gets killed : select a small enough number of records and delete them all together. It would be a solution if the table has primary key, otherwise (which is our case), the records that we select (e.g., by `select * from HeightOfTrees top 10000`) can't be used in the where clause of the deletion statement, because there they will present a larger set of data - keep in mind, nothing is unique, even the combination of `(tree_id, timestamp, height)`. We still face the risk to run out of log space. But how about not deleting them in the same transaction? In this way, we have to delete the records one by one, and each deletion requires a full scan of the table, which contains thousands of billions of records. It will take forever to get done.

Finally, I find a solution which does not take too much time and also keep the log space safe. The method is not complicated. Firstly, a count query is run to find out how many records are there in each month :
``` sql
select extract(year from timestamp) as year, extract(month from timestamp) as month, count(*) as count
from HeightOfTrees where tree_id in (SELECT tree_id FROM tree WHERE status='DOWN');
```

This needs one scan. If the `count` is smaller than a threshold, then delete records from this month directly. Otherwise, let `times=ceiling(count/threshold) + c` where `c` is a constant to make sure no surprising uneven distribution of the data. Assuming data is distributed linearly along the timestamp dimension, we can calculate the time frame for each deletion then do the delete accordingly. This still needs several days, but at least it is finite.

Lessons learned from this: **Always assign a primary key**.



