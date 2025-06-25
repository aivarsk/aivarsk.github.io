---
layout: post
title: Serializable isolation level and transaction processing
tags: tpc-c,serializable,oracle
---

While on the topic of [On-Line Transaction Processing Benchmarks](https://www.tpc.org/tpcc/results/tpcc_results5.asp), it's interesting to observe the strategies companies employ to achieve optimal results. All code for both the transaction monitor and database is available in the PDF report. Let's look at the [Oracle one that uses Oracle Tuxedo and the Oracle database](https://www.tpc.org/results/fdr/tpcc/Oracle_SPARC_SuperCluster_with_T3-4s_TPC-C_FDR_120210.pdf)

There's a lot of cryptic C code using the OCI interface and PL/SQL code blocks. But you won't find any signs of pessimistic (`SELECT ... FOR UPDATE`) or optimistic (`UPDATE .. WHERE version=:version`) locking. How so? How can they achieve correctness without explicit locking?

Oracle decided they could achieve the best results by using a serializable transaction isolation level:

```sql
ALTER SESSION SET ISOLATION_LEVEL = SERIALIZABLE
```

[Serializable isolation level](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/data-concurrency-and-consistency.html#GUID-8DA9A191-4CA3-4B1A-995F-4B17471C2738) in Oracle database means database will detect concurrent changes to table rows and return "ORA-08177: Cannot serialize access for this transaction". This is a bit similar to optimistic locking except the database doing it for you. And then you can add retries in your code that repeats all updates until success. Like this payment code from TPC-C:

```sql
DECLARE /* payz */
    not_serializable EXCEPTION;
    PRAGMA EXCEPTION_INIT(not_serializable,-8177);
    deadlock EXCEPTION;
    PRAGMA EXCEPTION_INIT(deadlock,-60);
    snapshot_too_old EXCEPTION;
    PRAGMA EXCEPTION_INIT(snapshot_too_old,-1555);
BEGIN
    LOOP BEGIN
       BEGIN 
           UPDATE cust
               SET c_balance = c_balance - :h_amount,
               c_ytd_payment = c_ytd_payment + :h_amount,
               c_payment_cnt = c_payment_cnt + 1
               WHERE ...;
           UPDATE dist
               SET d_ytd = d_ytd + :h_amount
               WHERE ...;
            EXIT;
        EXCEPTION WHEN not_serializable OR deadlock OR snapshot_too_old THEN
            ROLLBACK;
            :retry := :retry + 1;
        END;
    END LOOP;
END;
```

So I used this approach in the early version of an accounting system. But performance tests on hot accounts were so bad, I had to give up. The reasons for that are similar to what [Cliff Click shared about his experience with Hardware Transactional Memory](https://youtu.be/GEkeOHw87Sg?si=urUjkVKqeoaIscu7&t=944) and "perf counters" / "mod counters" leading to transaction retries.

With a serializable isolation level database is ignorant about what kind of changes your application makes. All updates are equally important for the database. But from the application point of view, a lot of changes are commutative and you don't care in what order they happen:

- incrementing transaction count
- increasing account balance
- decreasing account balance as long as it does not become negative

However, the database is not aware of that. When your transaction starts with the count of 42 and some other transaction manages to increment it before you, you get `ORA-08177` and have to retry again. The database sees that some bits have changed and that's all it cares about. But I don't care if the transaction count is 43, 44, or 89 after I increment it as long as the current (whatever) value is incremented by 1.

The only way how serializable transaction isolation level can be faster for TPC-C tests is when the contention is relatively low. For me, I settled on using relative updates and sometimes optimistic locking for accounts where the balance was decreased or used to calculate fees and interest based on the exact current value.  It was doing several entries per authorization along with cryptography, card checks, and network messages at [5,000 authorizations per second](https://blogs.oracle.com/solaris/post/tieto-card-suite-optimized-on-oracle-supercluster).
