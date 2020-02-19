# Snowflake Rules Based Grant Management

Simplifies the management of Snowflake grants, enables strict control and stops privileges falling into the wrong hands.

Allows security permissions in [Snowflake](https://www.snowflake.net) to be managed via rules that support wildcards and apply across all databases.

## UPDATE FEB 2019

A breaking change was made, switching from a single role per privilege definition, to a list of roles.
For existing users, rename all references to "Role" to "Roles", like so:

```json
"Role":"PUBLIC"
```

becomes

```json
"Roles":["PUBLIC"]
```

## How it works

### Default Snowflake behaviour

Snowflake provides the ability to grant collections of users (known as roles) privileges on either
specific objects, or all objects of a specific type, e.g.

* grant select on all tables in schema mydb.myschema to role analyst;
* grant select,insert,update,delete on table customer to role developer;

Nested roles are supported, which allows a hierarchy to be created.

### What this script provides in addition

This script provides a slightly different approach. Rather than using nested hierarchies to reduce
the number of GRANTs required, a rule file is provided which supports wildcards.

This means a very terse and readable configuration file can generate a very large number of GRANTs.

Because it's a structured file, it can reside in version control and can be subject to change management
enforcement (e.g. pull requests with mandatory reviews).

By default, the script will also remove any privileges not specified by the file, so any direct adding of
privileges into the database are revoked at each run.

### Restrictions

Currently, the following [GRANT clauses](https://docs.snowflake.net/manuals/sql-reference/sql/grant-privilege.html) are supported:

* accountObjectPrivileges, but only for databases and warehouse (resource monitors are left alone)
* schemaPrivileges
* schemaObjectPrivileges, but only for tables and views (stages, file formats, UDFs and sequences are left alone).

## Execution

With python 3 installed, the script can be ran like so:

```bash
pip install --upgrade snowflake-connector-python
python apply_permissions.py -a $SNOWFLAKE_ACCOUNT -u $SNOWFLAKE_USER -r $SNOWFLAKE_ROLE -w $SNOWFLAKE_WAREHOUSE --snowflake-region $SNOWFLAKE_REGION
```

It is expected that the environment variable `SNOWSQL_PWD` be set prior to calling the script, you should make this available to your build agent in some secure fashion.

You'll need to map between the branch name and the target environment name, e.g. master->prod

The user account will need the ability to manage grants. You can either use the existing SECURITYADMIN role or create another role and delegate like so:
```GRANT MANAGE GRANTS ON ACCOUNT TO ROLE "DEPLOYER";```

Or if you prefer docker, set the environment variables and run like so:

```bash
docker run -it --rm \
  -v "$PWD":/usr/src/applypermissions \
  -w /usr/src/applypermissions \
  -e SNOWFLAKE_ACCOUNT \
  -e SNOWFLAKE_USER \
  -e SNOWFLAKE_ROLE \
  -e SNOWFLAKE_WAREHOUSE \
  -e SNOWFLAKE_REGION \
  -e SNOWFLAKE_REGION \
  -e SNOWSQL_PWD \
  --name apply-permissions \
  python:3 /bin/bash -c "pip install --upgrade snowflake-connector-python && python apply_permissions.py -a $SNOWFLAKE_ACCOUNT -u $SNOWFLAKE_USER -r $SNOWFLAKE_ROLE -w $SNOWFLAKE_WAREHOUSE --snowflake-region $SNOWFLAKE_REGION"
```

### File specification

See below example.

Each accountObjectPrivileges item contains:

* **Purpose**: A plain english description of the rule
* **Role**: The name of the Snowflake role to grant permission to (specify exactly)
* Either **Databases** OR **Warehouses** : Matches the databases/warehouses to apply to. Format is [unix filename pattern](https://docs.python.org/2/library/fnmatch.html).
* **Privileges**: The [accountObjectPrivileges](https://docs.snowflake.net/manuals/sql-reference/sql/grant-privilege.html) to grant. (Must be possible or script will error)

Each schemaPrivileges item contains:

* **Purpose**: A plain english description of the rule
* **Role**: The name of the Snowflake role to grant permission to (specify exactly)
* **Databases**: Matches the databases to apply to. Format is [unix filename pattern](https://docs.python.org/2/library/fnmatch.html).
* **Schemas**: Matches the schemas to apply the rule to (uses the [Snowflake LIKE format](https://docs.snowflake.net/manuals/sql-reference/functions/like.html))
* **Privileges**: The [schemaPrivileges](https://docs.snowflake.net/manuals/sql-reference/sql/grant-privilege.html) to grant. (Must be possible or script will error)

Each schemaObjectPrivileges item contains:

* **Purpose**: A plain english description of the rule
* **Role**: The name of the Snowflake role to grant permission to (specify exactly)
* **Databases**: Matches the databases to apply to. Format is [unix filename pattern](https://docs.python.org/2/library/fnmatch.html).
* **Schemas**: Matches the schemas to apply the rule to (uses the [Snowflake LIKE format](https://docs.snowflake.net/manuals/sql-reference/functions/like.html))
* **Tables**: Matches the tables to apply the rule to (uses the [Snowflake LIKE format](https://docs.snowflake.net/manuals/sql-reference/functions/like.html)). Leave empty to not apply to any tables.
* **Views**: Matches the views to apply the rule to (uses the [Snowflake LIKE format](https://docs.snowflake.net/manuals/sql-reference/functions/like.html))Leave empty to not apply to any views.
* **FutureTables**: Set to true to apply these privileges to the specified roles all future tables in databases and schemas currently matching the pattern
* **FutureViews**: Set to true to apply these privileges to the specified roles all future views in databases and schemas currently matching the pattern
* **Privileges**: The [schemaObjectPrivileges](https://docs.snowflake.net/manuals/sql-reference/sql/grant-privilege.html) to grant. (Must be possible or it is ignored, e.g. UPDATE applied to a view)

#### Example file

```json
{
    "accountObjectPrivileges": [
        {
            "Purpose": "Grant usage level privileges to everyone on all databases",
            "Roles": ["PUBLIC"],
            "Databases": "*",
            "Privileges": ["USAGE"]
        },
        {
            "Purpose": "Grant usage on GENERAL_ANALYSIS to all users",
            "Roles": ["PUBLIC"],
            "Warehouses": "GENERAL_ANALYSIS",
            "Privileges": ["USAGE"]
        }
    ],
    "schemaPrivileges": [
        {
            "Purpose": "Preserve the demo database privileges",
            "Roles": ["PUBLIC"],
            "Databases": "DEMO_DB",
            "Schemas": "PUBLIC",
            "Privileges": ["MODIFY","CREATE FUNCTION","CREATE FILE FORMAT","CREATE PIPE","USAGE","CREATE STAGE","CREATE SEQUENCE","MONITOR","CREATE TABLE","CREATE VIEW"]
        }
    ],
    "schemaObjectPrivileges": [
        {
            "Purpose": "Grant data warehouse developers the complete freedom they need in the development environment",
            "Roles": ["DataWarehouseDevelopers"],
            "Databases": "*_DEV",
            "Schemas": "%",
            "Tables": "%",
            "Views": "%",
            "FutureTables": true,
            "FutureViews": true,
            "Privileges": ["SELECT","INSERT","UPDATE","DELETE","TRUNCATE"]
        },
        {
            "Purpose": "Grant reporting users the ability to select from any view in all production databases and schemas",
            "Roles": ["ReportViewers"],
            "Databases": "*_PROD",
            "Schemas": "%",
            "Tables": "",
            "Views": "%",
            "Privileges": ["SELECT"]
        },
        {
            "Purpose": "Grant customer service users the ability to select only from Customer-related views in production only",
            "Roles": ["CustomerService"],
            "Databases": "*_PROD",
            "Schemas": "%",
            "Tables": "CUSTOMER%",
            "Views": "%",
            "Privileges": ["SELECT"]
        }
    ]
}
```

## Notes

**Be very careful running this script if you have existing privileges that have already applied!**

To be safe, use the ```--grant-statements-file``` and ```--revoke-statements-file``` parameters to output the change scripts to a file, rather than apply them directly to the database.

This is a community-developed script, not an official Snowflake offering. It comes with no support.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details
