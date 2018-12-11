---
title: SF History in DWH
authors:
- val
tags:
- sf
- history
created_at: 2018-10-10 00:00:00
updated_at: 2018-10-10 16:25:27.549234
tldr: This is short description of the content and findings of the post.
---

**Contents**

[TOC]

_NOTE: this will include a table of contents when rendered on the site._



# SF History

We use two different approaches to do SF history:
* Using SF history objects
* Making our own

## Based on SF internal history objects

I don't store intermediate changes of the day, only the final ones.
I don't see any use case for these changes and it will be to many data points with increased maintenance.

Here I will use `Opportunity` entity as an example.

#### Key-Value table

The definition is in `db/sales/0050 - hst.sql`:
```sql
CREATE TABLE history_sales.salesforce_opportunity_history_kv
(
--just to see the date of creation
  dw_created_at  timestamp default current_timestamp,
--need this to ...
  dw_modified_at timestamp,
  opportunity_id text not null,
--used to determine the last version
  row_version    smallint,
--date of the change
  date_from      date not null,
--till what date the value was valid, if NULL means still valid
  date_to        date,
--field_name that got changed
  field          text,
  value          text,
  PRIMARY KEY (opportunity_id, field, row_version)
    DEFERRABLE,
--use this as a restriction that there is no duplicated changes & for "insert on update conflict" (merge)
  UNIQUE (opportunity_id, field, date_from)
);
```

ETL you can find in `db/sales/functions/history/0003 - sp_ext2history_salesforce_opportunity_history.sql`:

Params are: `fully_reload` (boolean)

1. Delete wrong history entries, with names of the SF users
2. Translate sub_stage__c values
3. Fill tmp table with rw as row version of the change
4. Fill salesforce_opportunity_history_kv table

#### Final history table

The definition is in `db/sales/0050 - hst.sql`:
```sql
--	here I put only the fields we are interested in
CREATE TABLE history_sales.salesforce_opportunity_history
(
  opportunity_id TEXT NOT NULL,
  date           DATE NOT NULL,
  stage_name     TEXT,
  owner_id       TEXT,
  sub_stage      TEXT,
  PRIMARY KEY (opportunity_id, date)
);
```

ETL you can find in `db/sales/functions/history/0004 - sp_ext2history_salesforce_opportunity_history_refill.sql`:

Params are: `p_year_int` - the year to refill; `p_quarter_int` - the quarter to refill to
If one is NULL, the filter on the field is not applied

In the function I left join history Key-Value table (`history_sales.salesforce_opportunity_history_kv`) to staging one(`stg_sales.salesforce_opportunity`) to have always all the opportunities there(`history_sales.salesforce_opportunity_history`).

History is propagated from created_date of an opportunity till the last_date, i.e. either date_to of the history table or finish_date. In case finish_date < date_from then take date_from. It happens when opportunity is closed but some new changes are coming, e.g. sub_stage got changed. If finish_date is empty I use current_date, means that an opportunity is still active or in process.

## Based on changes of ext_layer tables

Steps to implement history for a SF Entity, if there is none:

1. Create a table in `db/sales/0050 - hst.sql` with the proper definition
2. Mandatory fields are: `row_version`, `changes_list`, `till_systemmodstamp`, `id`, `systemmodstamp` and all the other are what you are interested in
3. Mark fields you want to track in `metadata.object_info` as setting the field `trackable` as true, the default values is true (for now)
4. Put in the corresponding execution plan you want to historize

Steps to track/untrack a field:

1. Mark/unmark fields you want(or don't) to track in `metadata.object_info` as setting the field `trackable`
4. Add/Drop definition(s) to a corresponding table in `db/sales/0050 - hst.sql`

Here I will use `User` entity as an example.

#### Changes table

The definition is in `db/sales/0050 - hst.sql`:
```sql
CREATE TABLE history_sales.salesforce_user
(
--	1 means the last one
  row_version             INT,
--	semicolon separated list of the changes,
--	e.g. ab_helper_rolename__c; max_discount__c; username;
  changes_list            TEXT,
--	till what date_time this change was effective
  till_systemmodstamp     TIMESTAMP,
-- SF ID of the record
  id                      TEXT,
-- SF system field that determines change date_time of the record
  systemmodstamp          TIMESTAMP,
--	some columns here
  CONSTRAINT salesforce_user_pkey PRIMARY KEY (id, row_version)
--	it needs to be Defferable, because during ETL the constraint violates
    DEFERRABLE
);
```

#### History function

There is a generic function to track history for any SF entity.
Params: `object_name_origin` - SF entity, lowercased; `object_group_origin` - object_group from the `metadata.object_info` table, to determine what tmp table to use for IDs, e.g. `primary`

The definition is in `db/sales/functions/history/0001 - sp_ext2history_salesforce.sql`.
A call looks like this:
```sql
SELECT history_sales.sp_ext2history_salesforce('account', 'primary');
```

I put all the comments in the file.

#### Final view

I've done it with materialized view, because I don't see any point in a proper ETL for this and it's much easier :).
I do very similar stuff as in the first approach with history.

The definition is in `db/sales/views/0001 - salesforce_user_history.sql`.
I put all the comments in the file.