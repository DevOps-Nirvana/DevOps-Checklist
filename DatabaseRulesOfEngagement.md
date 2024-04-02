# Decoupled and Low-Impact Database Schema Changes
If your organization isn’t doing this, you likely have (or will have) outages during/around deploys

Almost every service leverages a backend data store which is typically a SQL-based database (Oracle / MySQL / PostgreSQL).  For live systems it's critical to define “rules of engagement” for this database so you can maintain near 100% uptime and still allow an application to grow and expand over time.

To facilitate zero-downtime and provide flexibility and extensibility for the future, services will need to be able to handle schema changes automatically. In order to facilitate this our service needs the following.

1. A framework to track and version database changes which can be initiated before or after a deploy/rollback. Ideally, these "migrations" will become part of our codebase so we can track them along side our code. One such framework for could be​ ​https://flywaydb.org/​ or http://www.liquibase.org/​ or could be self-engineered.  Some modern frameworks and languages come with their own migration tools.
1. Backwards-compatible code (old code supports using new schema) OR forwards-compatible code (new code supports using old schema) but ideally BOTH so we can deploy and rollback seamlessly always, and have the schema changes completely decoupled from the code. Implications on how to do this are described here: https://spring.io/blog/2016/05/31/zero-downtime-deployment-with-a-database
1. Decoupled schema changes: execute schema changes needed for a deploy well before that deploy process, so only ​one​ thing changes at a time which we can track and quantify via metrics. If doing weekly/bi-weekly deploys, engineer this as part of the deployment process, eg: 3 days before deployment, roll out schema changes that are approved and pending.
1. Have all migrations tested both forwards and backwards, preferably automatically as part of a continuous integration process and CI/CD tool.
1. Have all schema changes vetted and approved by a DBA before going live.
1. Defined "Rules of Engagement" of allowed schema changes to a live system. Some recommended rules are…
1. Migrations must be atomic (as much as possible) and/or use transactions where necessary, and reduce unnecessary downtime by simplifying the statements.
    1. Eg: Do not have 3 separate queries on the same table for 3 separate alters, combine them into one statement.
```
# Instead of…
ALTER TABLE table ADD COLUMN col1 int;
ALTER TABLE table ALTER COLUMN col1 default '0';
ALTER TABLE table ADD COLUMN col2 int;
Simply…
ALTER TABLE table ADD COLUMN col1 int default '0', ADD COLUMN col2 int;
```
1. Always allowed to add a new Table
1. Rules of engagement to removing a table
1. ONLY if completely unused by new and old code
1. ONLY after three deployment cycles of table deprecation (or similar rule)
1. Typically before removing just do a table / database rename and let it “sit” on your database for another cycle or two just incase
1. Rules of engagement removing a column
    1. ONLY if completely unused by new and old code
    1. ONLY after three deployment cycles of column deprecation (or similar rule)
1. MUST CHECK the size of the table before running migration and/or attempt migration against snapshot of database to determine impact
    1. If a table is over x rows (eg: 1 million)
    1. If yes, then…
        1. May need to schedule downtime OR
        1. abandon this removal of column for planned downtime OR
        1. engineer and test an alternate zero-downtime "alter" scenario (specific tools and methods per database technology)
    1. If no (under x rows), OK
1. Rules of engagement adding a column
    1. REQUIRE all new columns must have a default value
    1. MUST CHECK the size of the table before running migration and/or attempt migration against snapshot of database to determine impact
1. IF checking size of table…
    1. If over x rows (eg: 1 million)
        1. may need to schedule downtime OR
        1. recommend making a new soft-FK'ed table to contain new columns for that table
    1. If under x rows, OK
1. IF attempting migration against a live database snapshot, if it takes more than x seconds (eg: 5 seconds) then require scheduled downtime for this migration and/or require the engineering team to alter the migration.
1. NOT allowed to…
    1. Rename a column
    1. Modify/Rename stored procedures
    1. Drop database (duh)
    1. etc
