# Decoupled and Low-Impact Database Schema Changes
If your organization isn’t doing this, you likely have (or will have) outages during/around deploys

tood: format me

Almost every service leverages a backend data store which is typically a SQL-based database (Oracle / MySQL / PostgreSQL).  For live systems it's critical to define “rules of engagement” for this database so you can maintain near 100% uptime and still allow an application to grow and expand over time.
To facilitate zero-downtime and provide flexibility and extensibility for the future, services will need to be able to handle schema changes automatically. In order to facilitate this our service needs the following.
A framework to track and version database changes which can be initiated before or after a deploy/rollback. Ideally, these "migrations" will become part of our codebase so we can track them along side our code. One such framework for could be​ ​https://flywaydb.org/​ or http://www.liquibase.org/​ or could be self-engineered.  Some modern frameworks and languages come with their own migration tools.
Backwards-compatible code (old code supports using new schema) OR forwards-compatible code (new code supports using old schema) but ideally BOTH so we can deploy and rollback seamlessly always, and have the schema changes completely decoupled from the code. Implications on how to do this are described here: https://spring.io/blog/2016/05/31/zero-downtime-deployment-with-a-database
Decoupled schema changes: execute schema changes needed for a deploy well before that deploy process, so only ​one​ thing changes at a time which we can track and quantify via metrics. If doing weekly/bi-weekly deploys, engineer this as part of the deployment process, eg: 3 days before deployment, roll out schema changes that are approved and pending.
Have all migrations tested both forwards and backwards, preferably automatically as part of a continuous integration process and CI/CD tool.
Have all schema changes vetted and approved by a DBA before going live.
Defined "Rules of Engagement" of allowed schema changes to a live system. Some recommended rules are…
Migrations must be atomic (as much as possible) and/or use transactions where necessary, and reduce unnecessary downtime by simplifying the statements.
Eg: Do not have 3 separate queries on the same table for 3 separate alters, combine them into one statement.
Instead of…
ALTER TABLE table ADD COLUMN col1 int;
ALTER TABLE table ALTER COLUMN col1 default '0';
ALTER TABLE table ADD COLUMN col2 int;
Simply…
ALTER TABLE table ADD COLUMN col1 int default '0', ADD COLUMN col2 int;
Always allowed to add a new Table
Rules of engagement to removing a table
ONLY if completely unused by new and old code
ONLY after three deployment cycles of table deprecation (or similar rule)
Typically before removing just do a table / database rename and let it “sit” on your database for another cycle or two just incase
Rules of engagement removing a column
ONLY if completely unused by new and old code
ONLY after three deployment cycles of column deprecation (or similar rule)
MUST CHECK the size of the table before running migration and/or attempt migration against snapshot of database to determine impact
If a table is over x rows (eg: 1 million)
If yes, then…
May need to schedule downtime OR
abandon this removal of column for planned downtime OR
engineer and test an alternate zero-downtime "alter" scenario (specific tools and methods per database technology)
If no (under x rows), OK
Rules of engagement adding a column
REQUIRE all new columns must have a default value
MUST CHECK the size of the table before running migration and/or attempt migration against snapshot of database to determine impact
IF checking size of table…
If over x rows (eg: 1 million)
may need to schedule downtime OR
recommend making a new soft-FK'ed table to contain new columns for that table
If under x rows, OK
IF attempting migration against a live database snapshot, if it takes more than x seconds (eg: 5 seconds) then require scheduled downtime for this migration and/or require the engineering team to alter the migration.
NOT allowed to…
Rename a column
Modify/Rename stored procedures
Drop database (duh)
etc
