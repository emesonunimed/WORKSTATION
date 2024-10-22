### VERIFICAR TABLESPACES

``` sql
SET LINESIZE 500
SET PAGESIZE 1000

SELECT Total.name AS "Tablespace Name",
    ROUND(NVL(Free_space, 0)) AS "Espaco Free",
    ROUND(NVL(total_space - Free_space, 0)) AS "Espaco usado",
    ROUND(total_space) AS "Espaco Total",
    ROUND(NVL(Free_space, 0) / total_space * 100) AS "% livre"
FROM 
    (SELECT tablespace_name, SUM(bytes / 1024 / 1024) AS Free_Space
    FROM sys.dba_free_space
    GROUP BY tablespace_name) Free,
    (SELECT b.name, SUM(bytes / 1024 / 1024) AS TOTAL_SPACE
    FROM sys.v_$datafile a, sys.v_$tablespace b
    WHERE a.ts# = b.ts#
    GROUP BY b.name) Total
WHERE Free.Tablespace_name(+) = Total.name
ORDER BY Total.name;
```

### VERIFICAR UMA TABLESPACE ESPECIFICO 

``` sql
SET LINESIZE 500
SET PAGESIZE 1000

SELECT Total.name AS "Tablespace Name",
       ROUND(NVL(Free_space, 0)) AS "Espaco Free MB",
       ROUND(NVL(total_space - Free_space, 0)) AS "Espaco usado MB",
       ROUND(total_space) AS "Espaco Total MB",
       ROUND(NVL(Free_space, 0) / total_space * 100) AS "% livre"
FROM
    (SELECT tablespace_name, SUM(bytes/1024/1024) AS Free_Space
    FROM sys.dba_free_space
    GROUP BY tablespace_name) Free
    LEFT JOIN
    (SELECT b.name, SUM(bytes/1024/1024) AS TOTAL_SPACE
    FROM sys.v_$datafile a, sys.v_$tablespace B
    WHERE a.ts# = b.ts#
    GROUP BY b.name) Total ON Free.Tablespace_name = Total.name
WHERE TABLESPACE_NAME = UPPER('&tablespace_name')
ORDER BY Total.name;
``` 

### VERIFICAR DATAFILES

``` sql
ALTER SESSION SET NLS_DATE_FORMAT = 'DD/MM/YYYY HH24:Mi:ss';

SET LINESIZE 500
SET PAGESIZE 1000
COLUMN NAME FORMAT A70

SELECT FILE#, NAME, TABLESPACE_NAME, CHECKPOINT_TIME, ROUND(SUM(BYTES) / 1024 / 1024) AS "SIZE MB"
FROM V$DATAFILE_HEADER
GROUP BY FILE#, NAME, CHECKPOINT_TIME, TABLESPACE_NAME
ORDER BY 2, 3;
``` 

### VERIFICAR UM DATAFILE ESPECIFICO

``` sql
ALTER SESSION SET NLS_DATE_FORMAT = 'DD/MM/YYYY HH24:Mi:ss';

SET LINESIZE 500
SET PAGESIZE 1000
COLUMN NAME FORMAT A70

SELECT FILE#, NAME, TABLESPACE_NAME, CHECKPOINT_TIME, ROUND(SUM(BYTES) / 1024 / 1024) AS "SIZE MB"
FROM V$DATAFILE_HEADER
WHERE TABLESPACE_NAME = UPPER('&TABLESPACE_NAME')
GROUP BY FILE#, NAME, CHECKPOINT_TIME, TABLESPACE_NAME
ORDER BY 2, 3;
```

### VERIFICAR UMA TABLESPACE E SEUS DATAFILES 

``` sql
SET SERVEROUTPUT ON;

DECLARE
    v_tablespace_name VARCHAR2(50) := '&tablespace_name';
BEGIN
    ### TABLESPACE INFO
    DBMS_OUTPUT.PUT_LINE('## TABLESPACE INFO ##');
    FOR rec IN (
        SELECT Total.name AS "TABLESPACE NAME",
               ROUND(NVL(Free_space, 0)) AS "ESPAÇO FREE",
               ROUND(NVL(total_space - Free_space, 0)) AS "ESPAÇO USADO",
               ROUND(total_space) AS "ESPAÇO TOTAL",
               ROUND(NVL(Free_space, 0) / total_space * 100) AS "% LIVRE"
        FROM (SELECT tablespace_name, SUM(BYTES / 1024 / 1024) AS Free_Space
              FROM sys.dba_free_space
              GROUP BY tablespace_name) Free,
             (SELECT b.name, SUM(BYTES / 1024 / 1024) AS TOTAL_SPACE
              FROM sys.v_$datafile a, sys.v_$tablespace b
              WHERE a.ts# = b.ts#
              GROUP BY b.name) Total
        WHERE Free.tablespace_name(+) = Total.name
        AND Total.name = UPPER(v_tablespace_name)
        ORDER BY Total.name
    ) LOOP
        DBMS_OUTPUT.PUT_LINE('TABLESPACE: ' || rec."TABLESPACE NAME");
        DBMS_OUTPUT.PUT_LINE('FREE: ' || rec."ESPAÇO FREE" || ' MB -' || rec."% LIVRE" || '%');
        DBMS_OUTPUT.PUT_LINE('USED: ' || rec."ESPAÇO USADO" || ' MB');
        DBMS_OUTPUT.PUT_LINE('TOTAL: ' || rec."ESPAÇO TOTAL" || ' MB');
    END LOOP;
```

    ### DATAFILE INFO

``` sql
    DBMS_OUTPUT.PUT_LINE('## DATAFILE INFO ##');
    FOR rec IN (
        SELECT FILE#, NAME, TABLESPACE_NAME, CHECKPOINT_TIME,
               ROUND(SUM(BYTES) / 1024 / 1024) AS "SIZE MB"
        FROM V$DATAFILE_HEADER
        WHERE TABLESPACE_NAME = UPPER(v_tablespace_name)
        GROUP BY FILE#, NAME, CHECKPOINT_TIME, TABLESPACE_NAME
        ORDER BY NAME, TABLESPACE_NAME
    ) LOOP
        DBMS_OUTPUT.PUT_LINE('NUMBER FILE: ' || rec.FILE#);
        DBMS_OUTPUT.PUT_LINE('NAME: ' || rec.NAME);
        DBMS_OUTPUT.PUT_LINE('CHECKPOINT: ' || rec.CHECKPOINT_TIME);
        DBMS_OUTPUT.PUT_LINE('TOTAL SIZE: ' || rec."SIZE MB" || ' MB');
    END LOOP;
END;
/
``` 

### VERIFICAR ESPAÇO NO ASM: 

``` sql
SELECT NAME, STATE, TOTAL_MB, FREE_MB FROM V$ASM_DISKGROUP;
``` 

### AUMENTAR TABLESPACE

``` sql
ALTER DATABASE DATAFILE <nº_datafile> RESIZE <tamanho_para_aumentar>;
``` 

### CRIAR DATAFILE 

``` sql
ALTER TABLESPACE <tablespace_name>
ADD DATAFILE <'/diretório/datafile_name.dbf'> SIZE 4000m AUTOEXTEND ON NEXT 500m MAXSIZE 31000m;
``` 

### VER TABELAS ASSOCIADAS A TABLESPACE.

``` sql
- POR NOME
SELECT TABLESPACE_NAME, TABLE_NAME 
FROM DBA_TABLES 
WHERE TABLESPACE_NAME = '&TABLESPACE_NAME';


SELECT SEGMENT_NAME, TABLESPACE_NAME 
FROM DBA_SEGMENTS 
WHERE SEGMENT_TYPE='TABLE' 
AND TABLESPACE_NAME='&TABLESPACE_NAME';
``` 

### VER TABLESPACE ASSOCIADA A UM PROPRIETÁRIO 

``` sql
SET LINESIZE 500
SET PAGESIZE 1000

SELECT 
    ds.OWNER AS "PROPRIETÁRIO",
    ds.SEGMENT_NAME AS "SEGMENT NAME",
    ds.TABLESPACE_NAME AS "TABLESPACE NAME",
    SUM(ds.BYTES) / 1024 / 1024 AS "TAMANHO TOTAL (MB)"
FROM 
    DBA_SEGMENTS ds
WHERE 
    ds.OWNER = UPPER('&OWNER')  ### Filtro pelo proprietário
GROUP BY 
    ds.OWNER, ds.SEGMENT_NAME, ds.TABLESPACE_NAME
ORDER BY 
    ds.TABLESPACE_NAME;
```

### VER ESPAÇO UNDO
``` sql
 SELECT size_allocated.tablespace_name,
         size_allocated.size_allocated_mb,
         size_used.size_used_mb,
         ROUND (
            size_used.size_used_mb / size_allocated.size_allocated_mb * 100,
            2)
            pct_size_used_mb
    FROM (  SELECT due.tablespace_name,
                   SUM (due.bytes) / 1024 / 1024 AS size_used_mb
              FROM dba_undo_extents due
          GROUP BY due.tablespace_name) size_used,
         (  SELECT dt.tablespace_name,
                   SUM (ddf.bytes) / 1024 / 1024 size_allocated_mb
              FROM dba_tablespaces dt, dba_data_files ddf
             WHERE     dt.tablespace_name = ddf.tablespace_name
                   AND dt.contents = 'UNDO'
          GROUP BY dt.tablespace_name) size_allocated
   WHERE size_allocated.tablespace_name = size_used.tablespace_name(+)
ORDER BY tablespace_name;
```

### VER INFORMAÇÕES TABLESPACE
``` sql
COLUMN TABLESPACE_NAME FORMAT A15
COLUMN SEGMENT_SPACE_MANAGEMENT FORMAT A10
COLUMN FLASHBACK_ON FORMAT A9
SELECT A.TABLESPACE_NAME,A.STATUS,A.EXTENT_MANAGEMENT,A.SEGMENT_SPACE_MANAGEMENT,B.FLASHBACK_ON "FLASHBACK" FROM DBA_TABLESPACES A
JOIN V$TABLESPACE B ON A.TABLESPACE_NAME = B.NAME;
```

### VER INFORMAÇÕES DETALHADAS ESPAÇO DAS TABLESPACES
``` sql
set linesize 300
set pagesize 8000
column dummy          noprint
column pct_used       format 999.9       heading "%|Used"
column name           format a25         heading "Tablespace Name"
column Mbytes         format 999,999,999 heading "MBytes" 
column Used_Mbytes    format 999,999,999 heading "Used|MBytes"
column Free_Mbytes    format 999,999,999 heading "Free|MBytes"
column Largest_Mbytes format 999,999,999 heading "Largest|MBytes"
column Max_Size       format 999,999,999 heading "MaxPoss|MBytes"
column pct_max_used   format 999.9       heading "%|Max|Used"
break   on  report 
compute sum of Mbytes      on report 
compute sum of Free_Mbytes on report 
compute sum of Used_Mbytes on report 
select ( select decode(extent_management,'LOCAL','*',' ') || 
                decode(segment_space_management,'AUTO','a ','m ')
        from dba_tablespaces
        where tablespace_name = b.tablespace_name
      ) || nvl(b.tablespace_name,nvl(a.tablespace_name,'UNKOWN')) name
      , Mbytes_alloc Mbytes
      , Mbytes_alloc-nvl(Mbytes_free,0) Used_Mbytes
      , nvl(Mbytes_free,0) Free_Mbytes
      , ((Mbytes_alloc-nvl(Mbytes_free,0))/Mbytes_alloc)*100 pct_used
      , nvl(Mbytes_largest,0) Largest_Mbytes
      , nvl(Mbytes_max,Mbytes_alloc) Max_Size
      , decode(Mbytes_max,0,0,(Mbytes_alloc/Mbytes_max)*100) pct_max_used
from ( select sum(bytes)/1024/1024   Mbytes_free
               , max(bytes)/1024/1024 Mbytes_largest
               , tablespace_name
       from  sys.dba_free_space
       group by tablespace_name
     ) a,
     ( select sum(bytes)/1024/1024      Mbytes_alloc
               , sum(maxbytes)/1024/1024 Mbytes_max
               , tablespace_name
       from sys.dba_data_files
       group by tablespace_name
       union all
       select sum(bytes)/1024/1024      Mbytes_alloc
               , sum(maxbytes)/1024/1024 Mbytes_max
               , tablespace_name
       from sys.dba_temp_files
       group by tablespace_name
     ) b
where a.tablespace_name (+) = b.tablespace_name
order by 1
/
```

###  VER TABELAS EXISTENTES A TABLESPACE
``` sql
SELECT TABLESPACE_NAME, OWNER, TABLE_NAME, STATUS FROM DBA_TABLES WHERE TABLESPACE_NAME='%TABLESPACE_NAME' ORDER BY TABLESPACE_NAME;
```

### VER TABLESPACE ENCRYPTED
``` sql
SELECT TABLESPACE_NAME, ENCRYPTED
FROM DBA_TABLESPACES;
```

### SEGUE ABAIXO UMA QUERY PARA DESCOBRIR OBJETOS POR TABLESPACE NO SEU ORACLE.
``` sql
SET PAGESIZE 120
BREAK ON TABLESPACE ON OWNER
COLUMN OBJECTS FORMAT A20
SELECT TABLESPACE_NAME, OWNER, COUNT(*)||' tables' Objects
FROM DBA_TABLES
GROUP BY TABLESPACE_NAME, OWNER
UNION SELECT TABLESPACE_NAME, OWNER, COUNT(*)||' indexes' Objects
FROM DBA_INDEXES
GROUP BY TABLESPACE_NAME, OWNER;
```
``` sql
SELECT TABLESPACE_NAME, OWNER, COUNT(*)||' tables' Objects
FROM DBA_TABLES
WHERE TABLESPACE_NAME = '[TABLESPACE_NAME]'
GROUP BY TABLESPACE_NAME, OWNER
UNION SELECT TABLESPACE_NAME, OWNER, COUNT(*)||' indexes' Objects
FROM DBA_INDEXES
WHERE TABLESPACE_NAME = '&TABLESPACE_NAME'
GROUP BY TABLESPACE_NAME, OWNER;
```

### VER SE TEM ALGUM USUARIO USANDO A TABLESPACE
``` sql
SELECT v.sid, v.serial#, s.USERNAME, s.SESSION_NUM, s.SESSION_ADDR, t.name
FROM V$SORT_USAGE s, v$session v , v$tablespace t
where v.SERIAL# = s.SESSION_NUM AND t.name = '&tablespace';
```

### VER OBJETOS QUE PERTECEM A UMA TABLESPACE
``` sql
SELECT TABLE_OWNER, TABLESPACE_NAME, TABLE_NAME AS "NOME DA TABELA", count(1) AS "QTDE. DE INDICES", INDEX_NAME AS "NOME DO INDEX"
FROM DBA_INDEXES 
WHERE OWNER='COREIT' 
GROUP BY TABLE_OWNER, TABLESPACE_NAME, TABLE_NAME, INDEX_NAME; 
```

### CONFIRMAR SE TEM OBJETOS ASSOCIADO A TABLESPACE
``` sql
SELECT TABLESPACE_NAME, OWNER, TABLE_NAME, STATUS 
FROM DBA_TABLES 
WHERE TABLESPACE_NAME='[TABLESPACE]]';
```

### OUTRA QUERY QUE LISTA OS OBJETOS POR TAMANHO.
``` sql
COL SEGMENT_NAME FORMAT A30;
COL OWNER FORMAT A30;
SELECT *
FROM (SELECT owner,segment_name||'~'||partition_name segment_name,bytes/(1024*1024) meg
FROM DBA_SEGMENTS 
WHERE TABLESPACE_NAME = '[TABLESPACE_NAME]'
ORDER BY  blocks desc);
```

### COLOCAR TABLESPACE OFFILE E ONLINE PARA OUTROS PROCEDIMENTOS
``` sql
ALTER TABLESPACE NAME_TABLESPACE OFFLINE NORMAL
ALTER TABLESPACE NAME_TABLESPACE ONLINE;
```
EXEMPLO: 
ALTER TABLESPACE EXAMPLE ONLINE;
ALTER TABLESPACE TESTEHR ONLINE;

### ADICIONAR DATAFILE
``` sql
ALTER TABLESPACE [TABLESPACE_NAME] ADD DATAFILE 'CAMINHO/DATAFILE/arquivo.dbf' SIZE [TAM_CRIADO] AUTOEXTEND OFF;
```
``` sql
ALTER TABLESPACE [TABLESPACE_NAME] ADD DATAFILE 'CAMINHO/DATAFILE/arquivo.dbf' SIZE [TAM_CRIADO] AUTOEXTEND ON NEXT TAM_INT_AUMENTO MAXSIZE 31G;
```

EXEMPLO ORACLE: 
ALTER TABLESPACE TS_VIAMAO_PROD ADD DATAFILE '/backup2/oracle/db/bdorcl/ts_viamao_prod51.dbf' SIZE 10g AUTOEXTEND ON NEXT 50M MAXSIZE 31G;
EXEMPLO ORACLE RAC: 
ALTER TABLESPACE TSD_CONSICO ADD DATAFILE '+DATA3' SIZE 10G AUTOEXTEND OFF; 

### CRIAR TABLESPACE
``` sql
CREATE TABLESPACE
[TABLESPACE_NAME]]
DATAFILE 'NOME_DATAFILE.dbf' SIZE 40M ONLINE;
```
EXEMPLO: 
``` sql
CREATE TABLESPACE DATA_TBS DATAFILE '/u02/oradata/ORCL2/datafile/mydb_datatbs_01.dbf' SIZE 100M;
```
-
###VER ESPAÇO TABLESPACE TEMPORARIA
``` sql
SELECT D.TABLESPACE_NAME, NVL (A.BYTES / 1024 / 1024, 0) SIZE_MB, NVL (T.BYTES / 1024 / 1024, 0) USED_MB, ROUND (NVL (T.BYTES / A.BYTES * 100, 0), 2) AS PCT_USED
FROM SYS.DBA_TABLESPACES D,
(SELECT TABLESPACE_NAME, SUM (BYTES) BYTES
FROM DBA_TEMP_FILES
GROUP BY TABLESPACE_NAME) A,
( SELECT SS.TABLESPACE_NAME,
SUM ((SS.USED_BLOCKS * TS.BLOCKSIZE)) BYTES
FROM GV$SORT_SEGMENT SS, SYS.TS$ TS
WHERE SS.TABLESPACE_NAME = TS.NAME
GROUP BY SS.TABLESPACE_NAME) T
WHERE D.TABLESPACE_NAME = A.TABLESPACE_NAME (+)
AND D.TABLESPACE_NAME = T.TABLESPACE_NAME (+)
AND D.EXTENT_MANAGEMENT LIKE 'LOCAL'
AND D.CONTENTS LIKE 'TEMPORARY'
ORDER BY 1;
```

### VER SESSÕES CONSUMIDO TABLESPACE TEMPORARIA
``` sql
SELECT d.tablespace_name, NVL (a.BYTES/1024/1024, 0) size_mb,
NVL (t.BYTES/1024/1024, 0) used_mb,
ROUND (NVL (t.BYTES / a.BYTES * 100, 0),2) AS pct_used
FROM SYS.dba_tablespaces d,
( SELECT tablespace_name, SUM (BYTES) BYTES
FROM dba_temp_files
GROUP BY tablespace_name) a,
( SELECT ss.tablespace_name,
SUM ((ss.used_blocks * ts.BLOCKSIZE)) BYTES
FROM gv$sort_segment ss, SYS.ts$ ts
WHERE ss.tablespace_name = ts.NAME
GROUP BY ss.tablespace_name) t
WHERE d.tablespace_name = a.tablespace_name(+)
AND d.tablespace_name = t.tablespace_name(+)
AND d.extent_management LIKE 'LOCAL'
AND d.CONTENTS LIKE 'TEMPORARY'
ORDER BY 1;
```

### Verificar quem está consumindo o espaço da tablespace temporaria 
``` sql
SELECT t1.inst_id, t1.username,t1.sql_id, t1.TABLESPACE, t1.segtype,
t2.SID, t1.session_num AS serial#,
SUM (t1.blocks * 8192/1024/1024) AS size_mb_temp
FROM gv$tempseg_usage t1, gv$session t2
WHERE t1.session_num = t2.inst_id
GROUP BY t1.inst_id, t1.username, t1.sql_id,
t1.TABLESPACE, t1.segtype, t2.SID, t1.session_num
ORDER BY size_mb_temp DESC;
```

###  VER SESSÕES CONSUMINDO A SEGMENTOS TEMPORARIA
``` sql
select se.sid, se.serial#, se.status, se.username, se.program, se.last_call_et, su.tablespace, su.blocks*8/1024 MB_Usado
from v$sort_usage su, v$session se
where se.saddr = su.SESSION_ADDR
order by su.blocks ;
 ```

###  VER TAMANHO DA TALESPACE TEMP
``` sql
SELECT d.tablespace_name, NVL (a.BYTES/1024/1024, 0) size_mb,
NVL (t.BYTES/1024/1024, 0) used_mb,
ROUND (NVL (t.BYTES / a.BYTES * 100, 0),2) AS pct_used
FROM SYS.dba_tablespaces d,
( SELECT tablespace_name, SUM (BYTES) BYTES
FROM dba_temp_files
GROUP BY tablespace_name) a,
( SELECT ss.tablespace_name,
SUM ((ss.used_blocks * ts.BLOCKSIZE)) BYTES
FROM gv$sort_segment ss, SYS.ts$ ts
WHERE ss.tablespace_name = ts.NAME
GROUP BY ss.tablespace_name) t
WHERE d.tablespace_name = a.tablespace_name(+)
AND d.tablespace_name = t.tablespace_name(+)
AND d.extent_management LIKE 'LOCAL'
AND d.CONTENTS LIKE 'TEMPORARY'
ORDER BY 1;
``` 

### VER SESSÃO USANDO UNDO
``` sql
set lines 200 pages 500
col USERNAME format a10
col SEGMENT_NAME format a15
col MACHINE format a10
col TABLESPACE_NAME format a15
col "RB NAME" format a20
col "SYSTEM PID" format a10
col SEGMENT_NAME format a20
SELECT r.name "RB NAME", p.pid "ORACLE PID", p.spid "SYSTEM PID ",
NVL (p.username, 'NO TRANSACTION') "OS USER", s.UserName, s.Status,
S.MACHINE,D.SEGMENT_NAME, D.BYTES/1024/1024 "MB" , D.BLOCKS, D.EXTENTS,
D.TABLESPACE_NAME
FROM v$lock l,
v$process p,
v$rollname r,
v$session s,
dba_segments D
WHERE l.sid = s.sid(+)
AND s.paddr = p.addr
AND TRUNC (l.id1(+)/65536) = r.usn
AND l.type(+) = 'TX'
AND l.lmode(+) = 6
AND R.NAME = D.segment_name
AND D.SEGMENT_TYPE in ('ROLLBACK','TYPE2 UNDO')
ORDER BY r.NAME
```

### VER TAMANHO DOS DADOS EXPIRADO E NÃO EXPIRADOS NO UNDO
``` sql
select 'EXPIRADO -> ' ||sum(bytes)/1024/1024 SITUACAO from dba_undo_extents
where status = 'EXPIRED'
UNION
select 'NÃO EXPIRADO -> ' ||sum(bytes)/1024/1024 SITUACAO from dba_undo_extents
where status = 'UNEXPIRED';
```
