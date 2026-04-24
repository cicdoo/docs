---
title: Integrating Odoo with MSSQL
deprecated: false
hidden: false
metadata:
  robots: index
---
We recently wired up an Odoo instance to pull data from a Microsoft SQL Server through pyodbc. The "right" answer is three lines in a Dockerfile — but we got there by falling into every pit along the way. This is what we learned.

# The challenge

Client wanted to integrate Odoo 19 with Microsoft SQL server using a non-standard python library called `pyodbc`, which requires additional debian packages to run such as `unixodbc` and since this is `MSSQL` there was no standard debian packages that could be installed without rebuilding and publishing our docker image.

# The solution

Our platform lets users declare extra system packages per instance through two free-text fields:&#x20;

- Odoo Debian Packages and&#x20;
- PostgreSQL Debian Packages.&#x20;

This allows installation of debian packages inside the relevant container whether Odoo or PostgreSQL on restart.

![](https://files.readme.io/ebabf665ff9cec414d1562bc6089ec61baeb80e928a4008e14d08de6ab4c4827-image.png)

The problem we faced was that there is no standard debian package for MSSQL in Debian Bookworm APT repositories, so we though of updating Dockerfile to include an extra layer to setup this package for the client.<br /><br />A difficult decision while the platform is flexible enough to deploy APT package but from standard repositories, so we did some research and found FreeTDS.<br /><br />FreeTDS is a set of libraries for Unix and Linux that allows your programs to natively talk to Microsoft SQL Server and Sybase databases that is available in Debian Bookworm APT repositories.

If you are integrating with MSSQL you can add those debian packages to Odoo container

```shell
unixodbc
unixodbc-dev
odbcinst
tdsodbc

```

Restart instance and in your custom module  you can connect to your `MSSQL` by constructing connection string using FreeTDS OBDC Driver.

```python
import pyodbc
conn = pyodbc.connect(
    "DRIVER={FreeTDS};"
    "SERVER=your.host;PORT=1433;"
    "DATABASE=yourdb;"
    "UID=user;PWD=pass;"
    "TDS_Version=7.4;"
)
```

<br />
