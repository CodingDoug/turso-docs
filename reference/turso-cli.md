---
description: Technical reference for the Turso CLI used to manage and access Turso databases.
keywords:
  - turso
  - database
  - cli
  - reference
---

# Turso CLI

The Turso CLI is the tool provided for managing Turso databases. If you are
getting started for the first time, we recommend following along with the
[getting started tutorial], which walks you through the process of installation,
authentication, creating a database, replicating it, querying it, and destroying
it.

## Conventions

The example commands on this page assume the following placeholders, expressed
as shell variables:

- `$GROUP_NAME`: The name of a [placement group] to work with
- `$DB_NAME`: The name of the database that was specified or assigned during
  creation.
- `$LOCATION_CODE`: A three-letter location code that indicate the physical
  location of a database instance. The command `turso db locations` outputs a
  full list of supported location codes.

## Installation

The Turso CLI has two installation options.

### Homebrew (macOS and Linux)

There is a Homebrew formula that's installed with the following command:

```bash
$ brew install tursodatabase/tap/turso
```

The formula includes an executable with autocompletion scripts for bash, fish,
and zsh.

### Scripted install

If you prefer the manage the installation directly, run the following command to
execute a shell script that downloads and installs the CLI:

```bash
$ curl -sSfL https://get.tur.so/install.sh | bash
```

The CLI is installed in a directory called `.turso` in your home directory. The
shell script attempts to add that to your shell’s PATH configuration. You must
start a new shell to see the change, or add it manually to the current shell.

### Verify the installation

Run the following command to make sure the Turso CLI is in your PATH:

```bash
$ turso --version
```

### Upgrade the CLI

If Homebrew was used to install the CLI, use the following commands to update
it:

```bash
$ brew update
$ brew upgrade
```

If you used the scripted install, use the CLI itself to update:

```bash
$ turso update
```

## Logging in to the CLI

The Turso CLI requires a GitHub account for authentication. You must log in to
be able to work with databases. All databases created while logged in with an
account belong to that account and are controlled by it. There is currently no
way to share database access with other accounts.

Use the command `turso auth login` to start the login process. The command
launches your default browser and prompts you to log in with GitHub. The first
time, you are asked to grant the GitHub Turso app some permissions to your
account. Accept this in order to continue. (If desired, you can revoke those
permissions later in the GitHub settings for your account.)

When the authentication process finishes, you are issued a user authentication
token. This token identifies your account to Turso. The token expires after one
week; afterward, you must log in again to get a new token.

:::info

The user auth tokens generated by the `turso auth login` command are different
in purpose from [database auth tokens for client access][database-auth-tokens]
and [platform API tokens](#platform-api-auth-tokens). They are not
interchangeable.

:::

### Running locally

If you are running the CLI on your local machine, the CLI receives this token as
part of the login flow and [stores it locally](#local-storage) for future use.
You can retrieve the persisted token string using:

```bash
$ turso auth token
```

### Running remotely

If you are running the CLI on a remote machine, it might not be able to launch a
browser. In that case, use the URL provided by `turso auth login` with a browser
you have access to in order to authenticate. The process ends with a page
showing your token. You can put this string in the environment variable
`TURSO_API_TOKEN` in a shell before running commands using a CLI that is not
logged in. For example:

```bash
$ export TURSO_API_TOKEN=[YOUR-TOKEN-STRING]
$ turso db locations
```

## Manage placement groups and logical databases

:::info

When working with [placement groups], note that they count toward the maximum
database allowance in your billing plan, even if you haven’t yet created a
database within the group. A placement group requires one database "unit" for
each location in the group, and you must have that capacity available in your
organization when you create a placement group or expand it to another location.

:::

### Create a placement group

To create a new placement group using a primary location with the lowest latency
to the machine where the command is run:

```bash
$ turso group create $GROUP_NAME
```

You can specify the location using the `--location` flag, providing the
location's three letter code.

:::note

It costs one database from your billing plan’s allowance to create a placement
group.

:::

:::note

Every placement group is assigned to an
[organization](#team-collaboration-with-organizations) which is used for
collaboration and billing. When you create a database, the CLI uses the current
organization as the container. By default, the CLI assumes a personal
organization with the same name as your GitHub username.

:::

### Create a logical database within a group

To create a new [logical database] with a random name in the named placement
group:

```bash
$ turso db create --group $GROUP_NAME
```

To specify the name of the database:

```bash
$ turso db create $DB_NAME --group $GROUP_NAME
```

If you omit the `--group` flag:

- If you have only one placement group, it is used.
- If you don't have any placement groups, one is created using the name
  "default".

#### Create a logical database using a SQLite database file

To create a new logical database and seed it with the contents of an existing
[SQLite3-compatible database file][sqlite3-db-file], use the `--from-file` flag:

```bash
$ turso db create $DB_NAME --group $GROUP_NAME --from-file $DB_FILE
```

### Replicate a database by adding a location to a group

You can replicate a logical database by adding a location to its placement
group. To add a location:

```bash
$ turso group locations add $GROUP_NAME $LOCATION_CODE
```

:::info

It costs one database from your billing plan’s allowance to add a location to a
placement group, no matter how many logical databases are contained within that
group.

:::

Adding a replica location to a group effectively replicates all logical
databases in that group, since they each share the same deployment and
replication behavior on the same hardware.

Client applications using a [logical database URL] are routed to the new
location if it's observed to have the lowest latency among all available
locations in the group.

Similarly, you can remove a replica location from a placement group:

```bash
$ turso group locations remove $GROUP_NAME $LOCATION_CODE
```

:::info

Removing a replica location is considered "safe" in that doesn't eliminate any
of the data in any logical database. The primary location always retains a copy
of everything.

:::

You can list the locations of a placement group using the `list` subcommand:

```bash
$ turso group locations list $GROUP_NAME
```

### Destroy a logical database

:::warning

Destruction of a logical database cannot be reversed. All copies of data are
deleted. The command prompts you to ensure this is really what you want.

:::

To destroy a logical database (from all locations, including the primary):

```bash
$ turso db destroy $DB_NAME
```

### Destroy a placement group

:::warning

Destroying a placement group permanently deletes all copies of all databases in
the group.

:::

To destroy an existing placement group:

```bash
$ turso group destroy $GROUP_NAME
```

### Update the libSQL server version of a placement group

To upgrade the version of libSQL server used for every logical database in a
placement group:

```bash
$ turso group update $GROUP_NAME
```

This command causes a brief moment of downtime for each instance as the update
happens. All existing connections are closed and must be reconnected. The libSQL
client libraries do this automatically.

To check the version of libSQL server for a logical database:

```bash
$ turso db show $DB_NAME
```

## Database client authentication tokens

[Client access] to query a Turso database from your application requires a
database authentication token. Treat any database token as a secret for use only
by your application backend.

:::info

The database auth tokens generated by the `turso db tokens create` command are
different in purpose from the user auth tokens received from [logging in to the
CLI](#logging-in-to-the-cli) and [platform API auth
tokens](#platform-api-auth-tokens). They are not interchangeable.

:::

### Creating a database token

To get a database auth token suitable for your app, run the following command:

```bash
$ turso db tokens create $DB_NAME
```

`$DB_NAME` is the name of your database. This command outputs an auth token
string that you can use to configure the libSQL client library wherever it
requires an "auth token" string.

Database tokens are not individually recorded by Turso. You may create as many
as you want. By default, these tokens never expire.

### Token expiration

If you want to put a limit on how much time a database token is valid, use the
`--expiration` flag to specify a duration in days. For example, for a 7-day
token, run the following:

```bash
$ turso db tokens create $DB_NAME --expiration 7d
```

### Read-only database tokens

To generate an auth token that has read-only access to the database.

```bash
$ turso db tokens create $DB_NAME --read-only
```

Read-only tokens enable a client to run queries with `select`, but disallow
`update`, `insert`, and `delete` commands.

### Invalidate database tokens

If a database token is ever leaked, you can invalidate all prior tokens for your
database with the following command:

```bash
$ turso db tokens invalidate $DB_NAME
```

This command restarts all of your database instances to use a new database token
signing key for any new tokens you create afterward.

## Team collaboration with organizations

Turso allows you to manage team database access and billing using a grouping
mechanism called "organizations". By default, the CLI assumes a personal
organization with the same name as your GitHub username.

### Create an organization

You can create a new organization using the CLI with the command:

```bash
$ turso org create $ORG_NAME
```

When the organization is created with the CLI:

- The organization is assigned a "slug" string which uniquely identifies it,
  based on the provided name.
- Your account is assigned the "owner" role. Only one owner per organization is
  supported.
- The new organization becomes the current organization for future CLI commands.
  You can [switch to another organization](#switch-organizations) as needed.

:::note

Organization slugs are globally unique. Creation of an organization fails in
case its assigned slug already exists.

:::

### List organizations

To list organizations of which you are the owner or a member:

```bash
$ turso org list
```

### Switch organizations

To change the current organization for future CLI commands, including those that
work with logical databases, provide its unique slug to the following command:

```bash
$ turso org switch $ORG_SLUG
```

The current organization is persisted in [local storage](#local-storage).

### Delete an organization

To delete an organization and all of its associated databases and billing
information:

```bash
$ turso org destroy $ORG_SLUG
```

Your personal organization can't be destroyed.

The CLI doesn't allow destroying an organization that is also the current
organization. You must [switch organizations](#switch-organizations) if
necessary.

### Manage members of an organization

Members of an organization are able to fully control all databases contained
within that organization. Only the owner is allow to access billing information.

To add a member to the current organization:

```bash
$ turso org members add $GITHUB_USERNAME
```

Use the `-a` flag to add with the admin role.

To list existing members:

```bash
$ turso org members list
```

To remove a member from the current organization:

```bash
$ turso org members rm $GITHUB_USERNAME
```

### Invite members to an organization

The owner of an organization can invite others to join. Sending an invitation
requires an email address. It can be any email address, not necessarily linked
to a GitHub account. To send the invitation:

```bash
$ turso org members invite $EMAIL
```

The recipient is asked to sign in to GitHub and accept the invitation to
complete the process. Turso then uses the GitHub account's address for further
email notifications.

Use the `-a` flag to invite with the admin role.

## Manage billing

The owner of an organization may manage billing information for the current
organization using the following commands:

| Command | Action |
| --- | --- |
| `turso plan show` | Shows the current payment plan and usage |
| `turso plan upgrade` | Upgrade the current plan (from starter to scaler) |
| `turso plan select` | Select a specific payment plan |
| `turso org billing` | Update credit card information |

A credit card is required to switch to the scaler plan. Credit card information
is entered on a per-organization basis. The CLI launches a web browser to manage
credit card information for the current organization.

See the website for [Turso pricing information].

## Platform API auth tokens

The Turso CLI can mint tokens for authentication with the [Turso Platform API].

:::info

The platform API auth tokens generated by the `turso auth api-tokens` command
are different in purpose from [database auth tokens for client
access][database-auth-tokens]. They are not interchangeable.

:::

### Mint a platform token

To mint a new token with a given name:

```bash
$ turso auth api-tokens mint $TOKEN_NAME
```

The CLI outputs the string value of the token.

:::warning

Be sure to copy the token to a safe place immediately after minting. The CLI
will not display its value again, and it is not recoverable.

:::

### List platform tokens

To list all tokens:

```bash
$ turso auth api-tokens list
```

### Revoke a platform token

To revoke a token with a given name:

```bash
$ turso auth api-tokens revoke $TOKEN_NAME
```

## Inspect database usage

Total database usage, measured for the purpose of [billing], is aggregated
across all [logical databases] associated with an account. Depending on your
payment plan, limits are applied to the following usage stats:

- Total amount of storage
- Number of rows read
- Number of rows written
- Total number of database instances (primary and replicas)
- Total number of database locations used

To see a summary for all databases for your account, use the following command:

```bash
$ turso plan show
```

You can get more detailed data about the current monthly usage of a single
logical database using the following command:

```bash
$ turso db inspect $DB_NAME
```

It might take up to one minute for the output to reflect recent operations.

You can show usage grouped per instance by running:

```bash
$ turso db inspect $DB_NAME --verbose
```

## Database dump and load

You can dump the entire contents of a Turso database using the following
command:

```bash
$ turso db shell $DB_NAME .dump > dumpfile.sql
```

The file `dumpfile.sql` contains all of the SQL commands necessary to rebuild
the database. libSQL and SQLite internal tables are not present.

:::note

`.dump` is a command you can run in the interactive shell, but you should
consider running it on the command line so its output can be easily saved in a
file.

:::

After creating a dump file, you can then load the dumped data into a new
database using the following command:

```bash
$ turso db shell $NEW_DB_NAME < dumpfile.sql
```

## Use libSQL server locally

The Turso CLI can invoke [libSQL server] for use during [local development]
instead of a managed Turso database. You must have the `sqld` binary in your
shell PATH. To start the server on port 8080, run:

```bash
$ turso dev
```

This starts libSQL server using an in-memory database. To persist data in a
SQLite database file, specify the path of the file:

```bash
$ turso dev --db-file path/to/db-file
```

The CLI outputs a URL you can use to connect to the local server. Use this URL
instead of the Turso database [libsql URL] when building locally. This URL can
be used with `turso db shell` and the libSQL client libraries.

You can change the client access port with the `--port` flag.

## Get help

The CLI offers help for all commands and subcommands. Run `turso help` to get a
list of all commands.  Use the `--help` flag to get help for a specific command
or subcommand. For example: `turso db --help`.

## Local storage

The CLI stores persistent data, including your auth token, in a file on your
computer. On macOS, the containing folder is `$HOME/Library/Application
Support/turso`. On Linux, it's `$HOME/.turso`. It's safe to delete this folder,
since it can be restored by logging in to the CLI again.


[Client access]: /libsql/client-access
[getting started tutorial]: /tutorials/get-started-turso-cli
[placement group]: /concepts#placement-group
[placement groups]: /concepts#placement-group
[location]: /concepts#location
[logical database]: /concepts#logical-database
[logical databases]: /concepts#logical-database
[logical database URL]: ./libsql-urls#logical-database-url
[replica]: /concepts#replica
[sqlite3-db-file]: https://www.sqlite.org/fileformat.html
[database-auth-tokens]: #database-client-authentication-tokens
[Turso Platform API]: /reference/platform-rest-api
[local development]: /reference/local-development
[libsql URL]: /reference/libsql-urls
[Turso pricing information]: https://turso.tech/pricing
[billing]: /billing-details
[libSQL server]: https://github.com/libsql/libsql#readme