+++
date = '2024-08-10'
draft = false
title = 'Prisma, PgBouncer & Prepared Statements'
+++

I recently had to use Prisma to interface with a PostgreSQL database that I wasn’t directly involved with at work, and I encountered an interesting problem.

We have several deployment environments, each with a hierarchy, and I never encountered the issue in the two lowest environments. However, from the third-tier environment upward, any of the three errors in the code block below randomly appear in the logs when the job that runs the database queries using the Prisma client is executed.

```
ConnectorError(ConnectorError { user_facing_error: None, kind: QueryError(PostgresError { code: "26000", message: "prepared statement \"s#\" does not exist", severity: "ERROR", detail: None, column: None, hint: None }), transient: false })

###############################################################################################################################################################################################################################################

ConnectorError(ConnectorError { user_facing_error: None, kind: QueryError(PostgresError { code: "42P05", message: "prepared statement \"s#\" already exists", severity: "ERROR", detail: None, column: None, hint: None }), transient: false })

###############################################################################################################################################################################################################################################

ConnectorError(ConnectorError { user_facing_error: None, kind: QueryError(PostgresError { code: "08P01", message: "bind message supplies # parameters, but prepared statement \"s#\" requires #", severity: "ERROR", detail: None, column: None, hint: None }), transient: false })
```

The “#” in “s#” represents a number. I’ve seen it go as high as 16 suggesting that it increases as the error occurs.

Turns out, PgBouncer which is a pretty popular connection pooler for PostgreSQL didn’t have support for prepared statements (which Prisma uses to execute its queries) until recently.

The fix is simple: you just need to add pgbouncer flag to your connection URL string as specified in the documentation:

```
postgresql://USER:PASSWORD@HOST:PORT/DATABASE
```

becomes

```
postgresql://USER:PASSWORD@HOST:PORT/DATABASE?pgbouncer=true
```

PS: I saw a [Prisma GitHub issue](https://github.com/prisma/prisma/issues/11643#issuecomment-1268341203) comment where the Supabase (if you use it in your stack) project needs to be restarted if you have initially connected without the pgbouncer flag.