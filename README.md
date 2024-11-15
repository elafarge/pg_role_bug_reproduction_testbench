# Postgres 14.14/15.9/16.5/17.1 bug repro

## The bug

In postgres version released a few hours ago, we've found a behavioral change
that impacts us a lot.

Users that are configured to assume a given role upon login with
`ALTER USER xxx SET ROLE yyy;` are not automatically switched to the given role
upon login any more.

This is particularly problematic for users of HashiCorp Vault's dynamic users,
who often rely on `ALTER ROLE xxx SET ROLE yyy` to make sure that dynamic &
short-lived users created by vault create postgres resources as a long-lived
role, and not as themselves.

We suspect [this commit](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=a5d2e6205)
to be the one that introduced this behavioral change.

## Reproduction

Using the attached docker compose file, we can easily reproduce the issue.

It deploys one postgres 15.8 instance (without the bug) and one posgtgres 15.9
instance (with the bug).

Launch it with `docker compose up -d`.

### Behavior on the old version

Connect to the old DB version with:

```shell
PGPASSWORD=password psql -h 127.0.0.1 -d postgres -p 7432 -U postgres
```

Then provision a role and a user for this role with

```sql
CREATE ROLE my_role;
CREATE USER my_user WITH PASSWORD 'password';
GRANT my_role TO my_user;
ALTER USER my_user SET ROLE my_role;
```

The last directive should ensure that upon login, `my_user`'s session role is
automatically switched to `my_role`.

We can confirm that by issuing

```sql
SELECT rolname, rolconfig FROM pg_roles WHERE rolname='my_user';

--  rolname |   rolconfig
-- ---------+----------------
--  my_user | {role=my_role}
-- (1 row)
```

Let's now log in as `my_user`...

```bash
PGPASSWORD=password psql -h 127.0.0.1 -d postgres -p 7432 -U my_user
```

And then confirm that the session's role is set to `my_role`:

```sql
SELECT current_user, session_user;

--  current_user | session_user
-- --------------+--------------
--  my_role      | my_user
-- (1 row)
```

### Behavior on the new version

Connect to the new instance version with

```shell
PGPASSWORD=password psql -h 127.0.0.1 -d postgres -p 8432 -U postgres
```

Then provision a role and a user for this role with

```sql
CREATE ROLE my_role;
CREATE USER my_user WITH PASSWORD 'password';
GRANT my_role TO my_user;
ALTER USER my_user SET ROLE my_role;
```

As we can see, the `rolcofig` column of the `pg_roles` view is set accordingly.

However, when logging in as `my_user`...

```bash
PGPASSWORD=password psql -h 127.0.0.1 -d postgres -p 8432 -U my_user
```

We can now see that the session's role hasn't been changed:

```sql
SELECT current_user, session_user;

--  current_user | session_user
-- --------------+--------------
--  my_user      | my_user
-- (1 row)
```
