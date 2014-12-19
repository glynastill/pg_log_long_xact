pg_log_long_xact
================

Pl/pgsql function to log long running transactions in PostgreSQL, can optionally output details of any blocking statements and cancel/terminate problematic statements/transactions.

Schedule with cron or similar if required.

Use the cancel/terminate functionality with care, and see notes about better alternatives for some use cases built into postgres.

Usage
---------

To simply log long running transactions just call the function passing the shortest duration to start logging at:

```sql
TEST=# SELECT public.pg_log_long_xact('1s');
NOTICE:  long_xact pid: 4465 duration: 431.325747 ms user: glyn application: psql client: [local] statement: <IDLE> in transaction
NOTICE:  long_xact pid: 16532 duration: 327.438302 ms user: glyn application: psql client: [local] statement: UPDATE balls SET description = 'TEST' WHERE id = 5;
 pg_log_long_xact
---------------
(0 rows)
```

To log extra information about locks the long running transactions might be waiting on, supply the second argument as true (although you could log similar information by just enabling log_lock_waits):

```sql
TEST=# SELECT public.pg_log_long_xact('1s', true);
NOTICE:  long_xact pid: 4465 duration: 471.885373 ms user: glyn application: psql client: [local] statement: <IDLE> in transaction
NOTICE:  long_xact pid: 16532 duration: 367.997928 ms user: glyn application: psql client: [local] statement: UPDATE balls SET description = 'TEST' WHERE id = 5;
NOTICE:  long_xact waiter pid: 16532 blocker detail is; pid: 4465 duration: 471.885373 ms relation: any (public.balls (RowExclusiveLock)) lock type: transaction id 311824482 user: glyn application: psql client: [local] statement: <IDLE> in transaction
 pg_log_long_xact
---------------
(0 rows)
```

To set the level of raise used pass a third argument which can be any of 'debug','log','info','notice','warning' or 'text' to output as a text resultset. 
(in hindsight the default should probably have been 'log')

```sql
TEST=# \t off
Showing only tuples.
TEST=# SELECT public.pg_log_long_xact('1s', true, 'text');
 long_xact pid: 4465 duration: 574.477076 ms user: glyn application: psql client: [local] statement: <IDLE> in transaction
 long_xact pid: 16532 duration: 470.589631 ms user: glyn application: psql client: [local] statement: UPDATE balls SET description = 'TEST' WHERE id = 5;
 long_xact waiter pid: 16532 blocker detail is; pid: 4465 duration: 574.477076 ms relation: any (public.balls (RowExclusiveLock)) lock type: transaction id 311824482 user: glyn application: psql client: [local] statement: <IDLE> in transaction
```

To make it start cancelling statements exceeding a specific duration we pass a duration as the fourth argument; the first transaction exceeding this will be cancelled 
on each run, with blocking statements prioritised.

 (As of pg 9.3 there's a lock_timeout parameter that will abort any statement waiting longer than the specified number of milliseconds, which is much better.  Note that the difference here is that this function will attempt to abort the blocking transaction rather than the waiting statement, or if there is no locking just the longest transaction.)

```sql
TEST=# SELECT public.pg_log_long_xact('1s', true, 'text', '10 minutes');
 long_xact pid: 4465 duration: 895.57988 ms user: glyn application: psql client: [local] statement: <IDLE> in transaction
 long_xact unable to cancel backend with pid: 4465
 long_xact pid: 16532 duration: 791.692435 ms user: glyn application: psql client: [local] statement: UPDATE balls SET description = 'TEST' WHERE id = 5;
 long_xact waiter pid: 16532 blocker detail is; pid: 4465 duration: 895.57988 ms relation: any (public.balls (RowExclusiveLock)) lock type: transaction id 31182
4482 user: glyn application: psql client: [local] statement: <IDLE> in transaction
```

We can also provide a duration as the fith argument to terminate backends with transactions exceeding this, with prioritisation as per cancelling:

```sql
TEST=# SELECT public.pg_log_long_xact('1s', true, 'text', '10 minutes', '15 minutes');
 long_xact pid: 4465 duration: 1026.90736 ms user: glyn application: psql client: [local] statement: <IDLE> in transaction
 long_xact terminated backend with pid: 4465
 long_xact pid: 16532 duration: 923.019915 ms user: glyn application: psql client: [local] statement: UPDATE balls SET description = 'TEST' WHERE id = 5;
```

By default the function only cancels/terminates the longest running blocker, or longest running transaction, we can be more forceful than that
by passing an extra parameter to tell it to cancel statements/terminate backlends of all long running transactions it finds:

```sql
TEST=#\t off
TEST=# SELECT public.pg_log_long_xact('1s', true, 'text', '2 minutes', '3 minutes', true);
 long_xact pid: 19065 duration: 187.279089 ms user: glyn application: psql client: [local] statement: <IDLE> in transaction
 long_xact terminated backend with pid: 19065
 long_xact pid: 16532 duration: 184.251569 ms user: glyn application: psql client: [local] statement: UPDATE balls SET description = 'TEST' WHERE id = 5;
 long_xact cancelled backend with pid: 16532
(4 rows)
```
