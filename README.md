# SQL-maintainence

Database Migration Implementation Plan with Rollback Strategy

Table of Contents
1. Introduction	1
2. Backup Existing Databases	1
3. Database Migration	2
4. Rollback Strategy	4
5. Conclusion	5


1. Introduction
This document outlines a plan for migrating databases in a SQL Server environment. The plan includes a rollback strategy to restore the original state in case of failure. The migration process involves detaching the databases, moving the data files, and then reattaching the databases. The rollback strategy involves restoring the databases from backup files.

2. Backup Existing Databases
Before starting the migration process, it's crucial to take a backup of all databases. This will allow us to restore the original state in case of failure. The following SQL script can be used to backup all user databases:

```sql
DECLARE @dbName NVARCHAR(128)
DECLARE @sql NVARCHAR(1000)
DECLARE @backupPath NVARCHAR(1000)

-- Specify the path where you want to store the backup files
SET @backupPath = 'C:\BackupPath\' -- Replace with your backup path

DECLARE db_cursor CURSOR FOR
SELECT name 
FROM sys.databases 
WHERE name NOT IN ('master', 'tempdb', 'model', 'msdb') -- Exclude system databases

OPEN db_cursor  
FETCH NEXT FROM db_cursor INTO @dbName  

WHILE @@FETCH_STATUS = 0  
BEGIN  
    SET @sql = 'BACKUP DATABASE [' + @dbName + '] TO DISK = N''' + @backupPath + @dbName + '.bak'' WITH NOFORMAT, NOINIT, NAME = N''' + @dbName + '-Full Database Backup'', SKIP, NOREWIND, NOUNLOAD, STATS = 10'
    EXEC sp_executesql @sql
    FETCH NEXT FROM db_cursor INTO @dbName  
END  

CLOSE db_cursor  
DEALLOCATE db_cursor
```

3. Database Migration
The migration process involves detaching the databases, moving the data files, and then reattaching the databases. The following SQL scripts can be used for these steps:

### Detach Databases
```sql
DECLARE @dbName NVARCHAR(128)
DECLARE @sql NVARCHAR(1000)

DECLARE db_cursor CURSOR FOR
SELECT name 
FROM sys.databases 
WHERE name NOT IN ('master', 'tempdb', 'model', 'msdb')

OPEN db_cursor  
FETCH NEXT FROM db_cursor INTO @dbName  

WHILE @@FETCH_STATUS = 0  
BEGIN  
    -- Set the database to offline to close existing connections
    SET @sql = 'ALTER DATABASE [' + @dbName + '] SET OFFLINE WITH ROLLBACK IMMEDIATE'
    EXEC sp_executesql @sql

    -- Detach the database
    SET @sql = 'EXEC sp_detach_db @dbname = ''' + @dbName + '''' 
    EXEC sp_executesql @sql

    FETCH NEXT FROM db_cursor INTO @dbName  
END  

CLOSE db_cursor  
DEALLOCATE db_cursor
```

### Move Data Files
After detaching the databases, move the data files (.mdf, .ndf, .ldf) to the new location.

### Attach Databases
```sql
-- Declare a table variable to hold the database names
DECLARE @dbNames TABLE (
    dbName NVARCHAR(128)
)

-- Insert the database names into the table variable
INSERT INTO @dbNames (dbName)
VALUES ('Database1'), ('Database2'), ('Database3') -- Add more names as needed

-- Declare variables to hold the current database name and the SQL command
DECLARE @dbName NVARCHAR(128)
DECLARE @sql NVARCHAR(1000)

-- Declare a cursor to iterate over the database names
DECLARE db_cursor CURSOR FOR


SELECT dbName FROM @dbNames

-- Open the cursor and fetch the first database name into the @dbName variable
OPEN db_cursor  
FETCH NEXT FROM db_cursor INTO @dbName  

-- Loop over all database names
WHILE @@FETCH_STATUS = 0  
BEGIN  
    -- Construct the SQL command to attach the current database
    SET @sql = 'CREATE DATABASE [' + @dbName + '] ON ( FILENAME = N''/path/to/your/mdf-data/file/' + @dbName + '.mdf'' ), ( FILENAME = N''/ path/to/your/ldf-log/file/' + @dbName + '_log.ldf'' ) FOR ATTACH'

    -- Execute the SQL command
    EXEC sp_executesql @sql

    -- Fetch the next database name into the @dbName variable
    FETCH NEXT FROM db_cursor INTO @dbName  
END  

-- Close and deallocate the cursor
CLOSE db_cursor  
DEALLOCATE db_cursor
```

4. Rollback Strategy
In case of failure during the migration process, the rollback strategy is to restore the databases from the backup files created at the beginning. The following SQL script can be used to restore all databases:

```sql
-- Declare a table variable to hold the database names
DECLARE @dbNames TABLE (
    dbName NVARCHAR(128)
)

-- Insert the database names into the table variable
INSERT INTO @dbNames (dbName)
VALUES ('Database1'), ('Database2'), ('Database3') -- Add more names as needed

-- Declare variables to hold the current database name and the SQL command
DECLARE @dbName NVARCHAR(128)
DECLARE @sql NVARCHAR(1000)

-- Declare a cursor to iterate over the database names
DECLARE db_cursor CURSOR FOR
SELECT dbName FROM @dbNames

-- Open the cursor and fetch the first database name into the @dbName variable
OPEN db_cursor  
FETCH NEXT FROM db_cursor INTO @dbName  

-- Loop over all database names
WHILE @@FETCH_STATUS = 0  
BEGIN  
    -- Construct the SQL command to restore the current database
    SET @sql = 'RESTORE DATABASE [' + @dbName + '] FROM DISK = N''C:\BackupPath\' + @dbName + '.bak'' WITH FILE = 1, MOVE N''' + @dbName + ''' TO N''C:\DataPath\' + @dbName + '.mdf'', MOVE N''' + @dbName + '_log'' TO N''C:\DataPath\' + @dbName + '_log.ldf'', NOUNLOAD, STATS = 5'

    -- Execute the SQL command
    EXEC sp_executesql @sql

    -- Fetch the next database name into the @dbName variable
    FETCH NEXT FROM db_cursor INTO @dbName  
END  

-- Close and deallocate the cursor
CLOSE db_cursor  
DEALLOCATE db_cursor
```

5. Conclusion
This plan provides a detailed process for migrating databases in a SQL Server environment, including a rollback strategy in case of failure. By following this plan, you can ensure that your databases are safely migrated with minimal downtime.

