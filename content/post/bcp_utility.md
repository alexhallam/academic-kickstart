---
title: "Bcp_utility"
date: 2020-02-23T16:25:59-05:00
draft: false
---

The `bcp` utility is the fastest way to push large data sets to Azure SQLDW.

The utility is very strict in how the data must look. If the data format
does not match the bcp command from the user then the SQL push will fail.

This is the bcp command I use.

`bcp DB.TABLE in data/forecast_to_push.csv -q -t, -F 1 -d forecast-dw -c -S forecast-db.database.windows.net -e logs/bcp.log -U username -P password`

The above string takes a csv file (forecast_to_push.csv) and uploads that file to the SQL table (DB.TABLE).

From the documentation the bcp command has the following structure and flags


```
bcp {dbtable | query} {in | out | queryout | format} datafile

  [-m maxerrors]            [-f formatfile]          [-e errfile]
  [-F firstrow]             [-L lastrow]             [-b batchsize]
  [-n native type]          [-c character type]      [-w wide character type]
  [-N keep non-text native] [-q quoted identifier]
  [-t field terminator]     [-r row terminator]
  [-a packetsize]           [-K application intent]
  [-S server name or DSN if -D provided]             [-D treat -S as DSN]
  [-U username]             [-P password]
  [-T trusted connection]   [-v version]             [-R regional enable]
  [-k keep null values]     [-E keep identity values]
  [-h "load hints"]         [-d database name]
```

I think most commands are self explanatory with one exception. The -t,
specifies that the file is delimited with commas. Many files are also written
as pipe delimited so the -t flag would need to be updated to -t|.

Also, it is common to not get the bcp push to work on the first try due to user
error. To quickly debug users errors I have found the -e flag very helpful. It
is an optional command which outputs an errfile to the specified path.

Even with the correct bcp string the push to the database may still fail. In my
experience this happens for one of two reasons:

1. The data was not written in the way bcp expected it

It is essential to make sure the data is written correctly and with the correct commands. write.table must be used. I have attempted using readr::write_csv() with no success. My guess is that write.table encodes data in a different way than write_csv(). The following command has worked well for me: write.table(forecast_to_push, full_path, row.names=FALSE, col.names = FALSE, sep = ",", quote = FALSE, na = "")

2. The datatypes in the source file do not match the data types in the destination table

The datatypes in the destination table may be checked in MSSQL with the following query: select COLUMN_NAME, DATA_TYPE from information_schema.columns where table_name = 'TABLE' and table_schema = 'DB'. These datatype erros also occure when the dimensions of the columns of the source file does not match the dimensions of the destination table.

## A Full Production Example

```
# connection string
connection_string <- DBI::dbConnect(odbc::odbc(),
                  Driver = "ODBC Driver 17 for SQL Server",
                  Server = "forecast-db.database.windows.net",
                  Database = "forecast-dw",
                  UID = "username",
                  PWD = "password")

# make sure that data is writen exaclty as shown here. bcp is very opinionated. The data must look perfect or else upload will fail.
full_path <- paste0("data_dir", "/", "forecast_to_push.csv")
write.table(forecast_to_push, full_path, row.names=FALSE, col.names = FALSE, sep = ",", quote = FALSE, na = "")

# drop table if NULL
DBI::dbExecute(connection_string, "IF OBJECT_ID('[forecast-dw].[DB].[TABLE]','U') IS NOT NULL
                    DROP TABLE [forecast-dw].[DB].[TABLE];")

# make empty table
DBI::dbExecute(connection_string, "
CREATE TABLE [forecast-dw].[DB].[TABLE]
(         id varchar(500)
        , data date
	, forecast_generation_date varchar(500)
	, forcast1 decimal(10, 3) NULL
	, forcast2 decimal(10, 3) NULL
	, forcast3 decimal(10, 3) NULL
)
")

# make bcp string
cmd_string_append <- paste0("/opt/mssql-tools/bin/bcp ", "DB.TABLE",
                            " in ", full_path,
                            " -q -t, -F 1 -d ",
                            "forecast-dw",
                            " -c -S ", "forecast-db.database.windows.net",
                            " -U ", "username",
                            " -P ", "password")

# print bcp string
message(cmd_string_append)

# execute the string as a command line command
system2(cmd_string_append)
```
