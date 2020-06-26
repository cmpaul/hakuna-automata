---
title: "MySQL Commands in Bash Scripts"
date: 2012-08-15T14:45:39-07:00
draft: false
---

My journey with trying to automate some routine MySQL setup. 

**tl;dr**: Escape your backticks when running MySQL commands in a bash script.

<!--more-->
---

I've been creating quite a few development environments for Alfresco lately for various projects -- extensions, dashlets, and new customizations -- and decided to try scripting the setup to save myself some time. This involves creating the directories and copying the necessary source and project files, which is simple enough, but it also includes creating an Alfresco database and granting the correct permissions to it. In order to do this from the command line (I'm using a bash script on Mac OS X), I had to figure out some escaping trickery to get this to work.

First some background: I use a naming convention for my Alfresco databases that is not entirely conventional. Specifically, I use names that begin with "alf_", followed by the project name, e.g., "alf_test-project". For the bash script to be able to execute the mysql command necessary, the underscore needs to be escaped. Normally, this can be done by executing the following SQL in an interactive MySQL terminal:

```sql
create database `alf_test-project`;
```

When executing this from the bash shell however, the back-ticks need to be escaped as follows, otherwise bash will direct the execution results of "alf_test-project" to that location in the command:

```bash
user=root
password=supersecretpassword
dbname=alf_test-project

mysql -u$user -p$password -e "create database \`$dbname\`;"
```

You can also do this with mysqladmin, avoiding the need to escape:

```bash
mysqladmin -u$user -p$password create $dbname
```

Next, the script assigns the correct permissions to the Alfresco database user so it can access the new database. Again, escaped back-ticks will ensure this command executes correctly:

```bash
mysql -u$user -p$password -e "GRANT CREATE ROUTINE, CREATE VIEW, ALTER, SHOW VIEW, CREATE, ALTER ROUTINE, EVENT, INSERT, SELECT, DELETE, TRIGGER, REFERENCES, UPDATE, DROP, EXECUTE, LOCK TABLES, CREATE TEMPORARY TABLES, INDEX ON \`$dbname\`.* TO '$user'@'localhost';"
```