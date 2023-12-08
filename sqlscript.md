# Collection of useful sql snippets
<details>
  <summary>Get the data size of all the tables within a sql database</summary>
  
```sql
SELECT 
    t.NAME AS TableName,
    s.Name AS SchemaName,
    p.rows AS RowCounts,
    SUM(a.total_pages) * 8 AS TotalSpaceKB, 
    SUM(a.used_pages) * 8 AS UsedSpaceKB, 
    (SUM(a.total_pages) - SUM(a.used_pages)) * 8 AS UnusedSpaceKB
FROM 
    sys.tables t
INNER JOIN      
    sys.indexes i ON t.OBJECT_ID = i.object_id
INNER JOIN 
    sys.partitions p ON i.object_id = p.OBJECT_ID AND i.index_id = p.index_id
INNER JOIN 
    sys.allocation_units a ON p.partition_id = a.container_id
LEFT OUTER JOIN 
    sys.schemas s ON t.schema_id = s.schema_id
WHERE 
    t.NAME NOT LIKE 'dt%' 
    AND t.is_ms_shipped = 0
    AND i.OBJECT_ID > 255 
GROUP BY 
    t.Name, s.Name, p.Rows
ORDER BY 
    5 desc

```
</details>

<details>
  <summary>Get the current executing query in SQL Database</summary>
  
```sql
SELECT DISTINCT r.session_id spid,
      r.blocking_session_id blkby, s.host_name, SUBSTRING(t.text, 1, 1000) SQLText, 
      ( SELECT TOP 1 SUBSTRING(t.text,stmt_start / 2, ( (CASE WHEN stmt_end = -1 THEN (LEN(CONVERT(nvarchar(max),t.text)) * 2) ELSE stmt_end END)  - stmt_start) / 2)  )  AS Text,
      r.Command, r.wait_time, r.wait_type, r.cpu_time, r.total_elapsed_time, r.total_elapsed_time/60000 inmins, 
      s.program_name, r.Status, s.Login_name
FROM dbo.sysprocesses P with (nolock) INNER JOIN sys.dm_exec_requests r(Nolock)
      ON p.SPId = r.Session_Id
      INNER JOIN sys.dm_exec_sessions s (Nolock)
      ON r.session_id = s.session_id
      CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE s.is_user_process = 1

```
</details>

<details>
  <summary>Get the current executing query in SQL Database along with Execution Plan</summary>
  
```sql
select a.session_id
	, a.blocking_session_id
	, a.command
	, b.[text]
	, db_name(a.database_id)
	, a.start_time
	, a.wait_time
	, a.last_wait_type
	, a.cpu_time
	, a.total_elapsed_time
	, a.reads, a.writes
	, c.query_plan
from sys.dm_exec_requests a
cross apply sys.dm_exec_sql_text(a.sql_handle) b
CROSS APPLY sys.dm_exec_query_plan(a.plan_handle) c
--where session_id <> @@SPID

```
</details>


<details>
  <summary>Get the status of the ongoing backup/restores</summary>
  
```sql
SELECT r.session_id,r.command,CONVERT(NUMERIC(6,2),r.percent_complete)
AS [Percent Complete],CONVERT(VARCHAR(20),DATEADD(ms,r.estimated_completion_time,GetDate()),20) AS [ETA Completion Time],
CONVERT(NUMERIC(10,2),r.total_elapsed_time/1000.0/60.0) AS [Elapsed Min],
CONVERT(NUMERIC(10,2),r.estimated_completion_time/1000.0/60.0) AS [ETA Min],
CONVERT(NUMERIC(10,2),r.estimated_completion_time/1000.0/60.0/60.0) AS [ETA Hours],
CONVERT(VARCHAR(1000),(SELECT SUBSTRING(text,r.statement_start_offset/2,
CASE WHEN r.statement_end_offset = -1 THEN 1000 ELSE (r.statement_end_offset-r.statement_start_offset)/2 END)
FROM sys.dm_exec_sql_text(r.sql_handle))) SQLText
FROM sys.dm_exec_requests r WHERE command IN ('RESTORE DATABASE','BACKUP DATABASE','restore log','BACKUP LOG')

```
</details>

<details>
  <summary>Get the stats of the table</summary>
  
```sql
SELECT
sp.stats_id, name, filter_definition, last_updated, rows, rows_sampled, steps, unfiltered_rows, modification_counter 
FROM sys.stats AS stat 
CROSS APPLY sys.dm_db_stats_properties(stat.object_id, stat.stats_id) AS sp
WHERE stat.object_id = object_id('<<TableName>>'); 

```
</details>

<details>
  <summary>Get the average fragmentation percentage of all the index in a table in database</summary>
  
```sql
SELECT a.index_id, name, avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats (DB_ID(N'<<Database Name>>'), OBJECT_ID(N'<<Table Name>>'), NULL, NULL, NULL) AS a
JOIN sys.indexes AS b ON a.object_id = b.object_id AND a.index_id = b.index_id; 

```
</details>

<details>
  <summary>Get the how the data is stored physically in the table</summary>
  
```sql
SELECT
      <<ColumnName>>
    ,sys.fn_PhysLocFormatter(%%physloc%%) AS PhysicalLocation
FROM <<TableName>> 

```
</details>


<details>
  <summary>Create user in PaaS database and provide access</summary>
  
```sql
CREATE USER [<<AAD User>>] FROM EXTERNAL PROVIDER
EXEC sp_addrolemember 'db_datareader','<<AAD User>>' 

```
</details> 


<details>
  <summary>Get the list of the roles available in the database</summary>
  
```sql
SELECT name, principal_id
FROM sys.database_principals 
WHERE
type = 'R'

```
</details>  


<details>
  <summary>Grant access to role on a specific object in the database</summary>
  
```sql
GRANT SELECT ON OBJECT::<<Object Name>> TO <<Role Name>>;

```
</details> 


<details>
  <summary>Get the last day of the month</summary>
  
```sql
--Last day of current month
SELECT EOMONTH(GETUTCDATE()) AS LastDayDate 

--Last day of next month
SELECT EOMONTH(GETUTCDATE(),1) AS LastDayDate 

--Last day of previous month
SELECT EOMONTH(GETUTCDATE(),-1) AS LastDayDate 

```
</details> 
	

<details>
  <summary>Get the object access of a role in SQL database</summary>
  
```sql
SELECT DB_NAME() AS DatabaseName
      ,DatabasePrincipals.name AS PrincipalName
      ,DatabasePrincipals.type_desc AS PrincipalType
      ,DatabasePrincipals2.name AS GrantedBy
      ,DatabasePermissions.permission_name AS Permission
      ,DatabasePermissions.state_desc AS StateDescription
      ,SCHEMA_NAME(SO.schema_id) AS SchemaName
      ,SO.Name AS ObjectName
      ,SO.type_desc AS ObjectType
FROM sys.database_permissions DatabasePermissions 
LEFT JOIN sys.objects SO ON DatabasePermissions.major_id = so.object_id 
LEFT JOIN sys.database_principals DatabasePrincipals ON DatabasePermissions.grantee_principal_id = DatabasePrincipals.principal_id 
LEFT JOIN sys.database_principals DatabasePrincipals2 ON DatabasePermissions.grantor_principal_id = DatabasePrincipals2.principal_id
WHERE DatabasePrincipals.name = 'Role Name' -- Change the Role Name


```
</details> 	
	


<details>
  <summary>Get the list of members who are having access to various roles</summary>
  
```sql
SELECT dp.name , us.name  
FROM sys.sysusers us right 
JOIN  sys.database_role_members rm ON us.uid = rm.member_principal_id
JOIN sys.database_principals dp ON rm.role_principal_id =  dp.principal_id


```
</details> 		
	
