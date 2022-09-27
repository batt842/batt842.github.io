---
layout: post
title: "Postgres invalid index"
date: 2022-08-16 00:00:00 +0900
categories: sw
description: postgres, index
comments: true
permalink: /posts/sw/3/not-published
published: false
---

Issue: index has been created but Postgres didn't use it.

> explain select * from alfred_transaction_history where platform_reference_id='test';
                                            QUERY PLAN
--------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..11088475.27 rows=1 width=122)
   Workers Planned: 2
   ->  Parallel Seq Scan on alfred_transaction_history  (cost=0.00..11087475.17 rows=1 width=122)
         Filter: ((platform_reference_id)::text = 'test'::text)
(4 rows)

The same query used the index properly.
> explain select * from alfred_transaction_history where platform_reference_id='test';
                                                               QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using idx_alfred_transaction_history_platform_reference_id on alfred_transaction_history  (cost=0.57..8.59 rows=1 width=130)
   Index Cond: ((platform_reference_id)::text = 'test'::text)
(2 rows)

Checked the table size => both are M-scale

Maybe Postgres mis-evaluated the cost of different query plans. Let's disable seq scan.
BEGIN;
set local enable_seqscan=off;
explain select * from alfred_transaction_history where platform_reference_id='test';
COMMIT;
                                           QUERY PLAN
------------------------------------------------------------------------------------------------
 Seq Scan on alfred_transaction_history  (cost=10000000000.00..10014317721.80 rows=1 width=122)
   Filter: ((platform_reference_id)::text = 'test'::text)
(2 rows)

It was even worse. It was more posibile that something happened to the index and Postgres couldn't use it instead of not willing to use it.

But how?
Size doesn't look weird.
SELECT pg_size_pretty (pg_relation_size('idx_alfred_transaction_history_platform_reference_id'));

Found it. The index was invalid!
alfred-fp-tw=> \d+ alfred_transaction_history
                                                          Table "public.alfred_transaction_history"
         Column         |            Type             | Collation | Nullable |                Default                | Storage  | Stats target | Description
------------------------+-----------------------------+-----------+----------+---------------------------------------+----------+--------------+-------------
 id                     | bigint                      |           | not null | get_id('seq_trans_history'::regclass) | plain    |              |
 creation_datetime      | timestamp without time zone |           | not null |                                       | plain    |              |
 description            | character varying(150)      |           | not null |                                       | extended |              |
 platform_reference_id  | character varying(255)      |           | not null |                                       | extended |              |
 currency               | character varying(8)        |           | not null |                                       | extended |              |
 amount                 | numeric(36,16)              |           | not null |                                       | main     |              |
 transaction_datetime   | timestamp without time zone |           | not null |                                       | plain    |              |
 transaction_owner_id   | bigint                      |           |          |                                       | plain    |              |
 transaction_owner_type | character varying(64)       |           |          |                                       | extended |              |
 event_type             | character varying(64)       |           | not null |                                       | extended |              |
 wallet_id              | bigint                      |           | not null |                                       | plain    |              |
 update_datetime        | timestamp without time zone |           |          | now()                                 | plain    |              |
 expiration_datetime    | timestamp without time zone |           |          |                                       | plain    |              |
Indexes:
    "alfred_transaction_history_pkey" PRIMARY KEY, btree (id)
    "idx_alfred_transaction_history_platform_reference_id" btree (platform_reference_id) INVALID
    "idx_alfred_transaction_history_update_datetime" btree (update_datetime)
    "idx_trans_hist_wallet_creation" btree (wallet_id, creation_datetime DESC)
Check constraints:
    "alfred_transaction_history_not_null_update_datetime" CHECK (update_datetime IS NOT NULL) NOT VALID
Foreign-key constraints:
    "fk_tx_hist_wallet_id" FOREIGN KEY (wallet_id) REFERENCES wallet_wallet(id)
Referenced by:
    TABLE "alfred_purchase_breakdown" CONSTRAINT "fk_purch_brk_tx_hist" FOREIGN KEY (transaction_history_item_id) REFERENCES alfred_transaction_history(id)
Triggers:
    trigger_set_update_datetime BEFORE UPDATE ON alfred_transaction_history FOR EACH ROW EXECUTE PROCEDURE set_update_datetime()

I recalled it...
maybe the previous index creation job was interrupted and cancelled 

to detect,
SELECT * FROM pg_class, pg_index WHERE pg_index.indisvalid = false AND pg_index.indexrelid = pg_class.oid;

P-SE O