---
layout: post
title:  "Analyzing covid19 data with SQL"
---

RIVM is a governmental health organisation in the Netherlands that publishes data here: https://data.rivm.nl/covid-19/. I show how the data can be inserted in a database for analysis.

Tools used: PostgreSQL (the database system), HeidiSQL (a postgresql client application) and the Anaconda Python distribution including pandas and sqlalchemy (to extract data from sources and insert it into the database).

### Installation

Installer for PostgreSQL: https://www.enterprisedb.com/downloads/postgres-postgresql-downloads  
Installer for Anaconda: https://www.anaconda.com/products/individual  
Installer for HeidiSQL: https://www.heidisql.com/download.php

#### PostgreSQL

During installation of PostgreSQL it doesn't matter how strong the superuser password is for this purpose. The password is important if you want to make the database accessible from the internet or have sensitive data in it.

After the installer finishes it asks if you want to install additional tools. You don't need them.

#### Anaconda

During installation of Anaconda I choose to install it in `D:\anaconda3` because that's simple and doesn't contain spaces. It's usually a good idea to avoid spaces in paths for programming because they cause annoying problems. I avoid them just to be sure.

After installation you need to add a few directories to the PATH environment variable to make python accessible from the commandline. You can use the User PATH variable or the system PATH variable. It doesn't really matter if you are the only user of the computer. How to change environment variables: https://superuser.com/questions/949560/how-do-i-set-system-environment-variables-in-windows-10

The directories to be added are `D:\anaconda3`, `D:\anaconda3\scripts` and `D:\anaconda3\Library\bin` separated by `;`. For example, if the content of the variable is `C:\Example1\bin;C:\Example2;` right now, then it needs to be changed to: `D:\anaconda3;D:\anaconda3\scripts;D:\anaconda3\Library\bin;C:\Example1\bin;C:\Example2;`

After that you can install the Python package psycopg2 which is needed to work with a PostgreSQL database. Find the Anaconda command prompt and execute the command: `conda install psycopg2`. Installing other packages works the same way. You can also use pip3 like with a normal Python installation (not Anaconda) by executing the command `pip3 install psycopg2` from a normal command prompt.

After that you can test if everything works by starting python with the command `python` and then while in python entering `import pandas`, `import sqlalchemy` and `import psycopg2`:

![image](https://user-images.githubusercontent.com/29734312/94369908-c3ab5700-00ec-11eb-87a6-00a966d49cca.png)

#### HeidiSQL

Installation is straightforward.

### Connecting to the database

There is a default database called postgres. Start HeidiSQL and see if you can connect to the database. HeidiSQL starts with the session manager. This is the configuration you need:

![image](https://user-images.githubusercontent.com/29734312/94370002-6532a880-00ed-11eb-9123-5aed9b2ce3f1.png)

After that you can test if the database responds to queries by executing a simple query like `select 1` in the Query tab. You can always create more such tabs with Ctrl+T.

![image](https://user-images.githubusercontent.com/29734312/94370077-015caf80-00ee-11eb-8e0e-e9aecc1df9ea.png)

### Inserting data into the database

#### First: without worrying about the details yet

PostgreSQL has schemas. A table always belongs to a schema. Rather than using the default public schema, I like to create a separate schema in the database for the data, called data, so I excute this in HeidiSQL:

```sql
create schema data
```

For example I use the file COVID-19_casus_landelijk.json from https://data.rivm.nl/covid-19/.

Anaconda has installed the Python development environment Spyder. Start Spyder and create a new file. Save it somewhere and make sure the python file is in the same directory as the data. Then you can use this python script to insert the data into the database, where `yourpassword` needs to be replaced with your actual password:

```py
import json
import pandas as pd
from sqlalchemy import create_engine

database = "postgres"
schema = "data"
engine = create_engine("postgresql://postgres:yourpassword@localhost/" + database)

dataDict = None
with open("COVID-19_casus_landelijk.json", "r", encoding = "utf-8") as infile:
    dataDict = json.load(infile)

df = pd.DataFrame(data = dataDict)

df.to_sql("casus_landelijk", engine, schema, if_exists = "replace")
```

It opens the file, then json.load creates a native Python dictionary from the JSON-string in the file, then the dictionary is used as the source for a pandas dataframe (which is like a table), which is then inserted in the database. `casus_landelijk` is the name of the table in the database. `if_exists = "replace"` causes the table in the database to be deleted and recreated when the script is executed again.

This is what it looks like in Spyder:

![image](https://user-images.githubusercontent.com/29734312/94370514-cb6cfa80-00f0-11eb-8812-c99d6fafc4dc.png)

After executing that you can check if you can see the data in HeidiSQL by executing the query `select * from data.casus_landelijk`.

You can also use the GUI: first click the postgres database with its elephant icon on the left and press f5 to refresh the objects and schemas shown by HeidiSQL. Open the data schema, click the table called casus_landelijk and click the data tab. Then you see this:

![image](https://user-images.githubusercontent.com/29734312/94371017-9a41f980-00f3-11eb-882b-e5c2704c8504.png)

#### Second: the details

The simple procedure above doesn't set the right datatypes and the column names contain uppercase characters. The uppercase characters require you to use quotes around the names when writing queries, and the wrong datatypes make it harder to perform calculations. Unfortunately the solution is specific to this particular JSON file. All data sources have their own weird problems. If the column names contained spaces, for example, those would have had to be replaced too.

To convert the column names to lowercase, this can be done in python before insertion into the database happens:

```py
df.columns = [col.lower() for col in df.columns]
```

and this is a query to set the datatypes:

```sql
alter table data.casus_landelijk
	alter column date_file type timestamp using date_file::timestamp
	,alter column date_statistics type date using date_statistics::date
;
```

Having to execute the query by hand every time the data is refreshed is annoying, so I let Python execute it. The final script then becomes:

```py
import json
import pandas as pd
from sqlalchemy import create_engine

database = "postgres"
schema = "data"
engine = create_engine("postgresql://postgres:admin@localhost/" + database)

dataDict = None
with open("COVID-19_casus_landelijk.json", "r", encoding = "utf-8") as infile:
    dataDict = json.load(infile)

df = pd.DataFrame(data = dataDict)
df.columns = [col.lower() for col in df.columns]

df.to_sql("casus_landelijk", engine, schema, if_exists = "replace")

engine.execute(
    "alter table " + schema + ".casus_landelijk \n"
	+ "alter column date_file type timestamp using date_file::timestamp \n"
	+ ",alter column date_statistics type date using date_statistics::date \n"
    + "; \n"
)
```

Having the literal query text in the code like that is error prone and harder to read so you can also store it in a file, then let Python read the file to obtain the query, if you want.

### Analyzing the data

Now the data can be analyzed by writing and executing queries in HeidiSQL. For example:

```sql
with q as (
	select
		c.agegroup
		,sum(
			case c.deceased
				when 'Yes' then 1
				else 0
			end
		) as deceased
		,count(*) cases
	from
		data.casus_landelijk c
	group by
		c.agegroup
)

select
	q.*
	,deceased::double precision / cases as mortality
from
	q
order by
	q.agegroup
```

for the mortality per age group.

![image](https://user-images.githubusercontent.com/29734312/94371500-43d6ba00-00f7-11eb-9cfc-62463482b0de.png)

### Why I chose this way of working

- Many other data sources can be converted to pandas dataframes and inserted into the database in a similar way.
- The creation of the database is repeatable.
- I like SQL because it's a powerful language designed for creating tables.
- Having all the data in the database means having access to all of that data at once while using SQL.
