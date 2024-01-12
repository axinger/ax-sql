# ax-sql
SQL语句

## 查询当前数据及上一个数据
### 上一条
```mysql
SELECT
    t.*,
    LAG(t.id) OVER (PARTITION BY t.code ORDER BY t.id) as prd_id,
    LAG(t.code) OVER (PARTITION BY t.code ORDER BY t.id) as prd_code,
    LAG(t.quantity) OVER (PARTITION BY t.code ORDER BY t.id) as pre_quantity
FROM 
    timeline t
ORDER BY t.id DESC;
```
### 下一条,t.id DESC) 关键点
```mysql
SELECT
    t.*,
    LAG(t.id) OVER (PARTITION BY t.code ORDER BY t.id DESC) as prd_id,
    LAG(t.code) OVER (PARTITION BY t.code ORDER BY t.id DESC) as prd_code,
    LAG(t.quantity) OVER (PARTITION BY t.code ORDER BY t.id DESC) as pre_quantity
FROM 
    timeline t
ORDER BY t.id;

```
```sql
WITH RankedData AS (
    SELECT   [机台编号] AS [上次机台编号],
    [单据编号] AS [上次单据编号],
    [任务下发时间] AS [上次任务下发时间],
    [计划开始时间] AS [上次计划开始时间],
    [计划结束时间] AS [上次计划结束时间],
    [维护开始时间] AS [上次维护开始时间],
    [维护结束时间] AS [上次维护结束时间],
    ROW_NUMBER ( ) OVER ( PARTITION BY [机台编号] ORDER BY [任务下发时间] DESC ) AS RowNum
FROM
    ODS_CY_MES_T_EMWO
    ) SELECT
          t.*,
          preData1.*
      FROM
          ODS_CY_MES_T_EMWO t
              LEFT JOIN RankedData preData1 ON t.[机台编号] = preData1.[上次机台编号]
              AND preData1.RowNum = 2
      WHERE
          t.[任务下发时间] >= '2022-12-01 00:00:00'
      ORDER BY
          t.[任务下发时间] DESC;
```
## 插入当前数据及上一个数据,不能用上面方法
```sql
DECLARE @pDATA_DT DATETIME
DECLARE @pROWCOUNT INT

SET @pDATA_DT = GETDATE();

INSERT INTO [dbo].[DWS_ST_T_MES_EMWO]
            (
                [DATA_DT],
                [FYEAR],
                [FMONTH],
                [F_YMD],
                [FMONTH_PRE],
                [机台名称],
                [机台编号],
                [任务下发时间],
                [计划开始时间],
                [计划结束时间],
                [维护开始时间],
                [维护结束时间],
                [持续时长],
                [维修时长],
                [描述],
                [单据编号],
                [处理人],
                [状态],
                [上次单据编号],
                [上次任务下发时间],
                [上次计划开始时间],
                [上次计划结束时间],
                [上次维护开始时间],
                [上次维护结束时间]
            )
--设备名称、设备编号、故障开始时间、持续时长、内容、当前责任人、状态
SELECT
    YEAR(t.[任务下发时间]) AS FYEAR,
    YEAR(t.[任务下发时间]) * 100 + MONTH(t.[任务下发时间]) AS FMONTH,
    CONVERT(VARCHAR, t.[任务下发时间], 112) AS F_YMD,
    YEAR(DATEADD(MONTH, -1, t.[任务下发时间])) * 100 + MONTH(DATEADD(MONTH, -1, t.[任务下发时间])) AS FMONTH_PRE,
    t.[机台名称],
    t.[机台编号],
    t.[任务下发时间],
    t.[计划开始时间],
    t.[计划结束时间],
    t.[维护开始时间],
    t.[维护结束时间],
    DATEDIFF(MINUTE, ISNULL(t.[任务下发时间], GETDATE()), ISNULL(t.[维护结束时间], GETDATE())) AS [持续时长],
    DATEDIFF(MINUTE, ISNULL(t.[维护开始时间], GETDATE()), ISNULL(t.[维护结束时间], GETDATE())) AS [维修时长],
    t.[描述],
    t.[单据编号],
    t.[处理人],
    CASE
        WHEN t.[维护开始时间] IS NULL THEN '待处理'
        WHEN t.[维护开始时间] IS NOT NULL AND t.[维护结束时间] IS NULL THEN '处理中'
        ELSE '已处理'
    END AS [状态],
	preData1.*
FROM
    ODS_CY_MES_T_EMWO t OUTER APPLY (
    SELECT TOP
    1 [单据编号] AS [上次单据编号],
    [任务下发时间] AS [上次任务下发时间],
    [计划开始时间] AS [上次计划开始时间],
    [计划结束时间] AS [上次计划结束时间],
    [维护开始时间] AS [上次维护开始时间],
    [维护结束时间] AS [上次维护结束时间]
    FROM
    ODS_CY_MES_T_EMWO pre
    WHERE
    t.[机台编号] = pre.[机台编号]
    AND pre.[任务下发时间] < t.[任务下发时间]
    ORDER BY
    [任务下发时间] DESC
    ) AS preData1
WHERE
    t.[任务下发时间] >= '2022-12-01 00:00:00'
ORDER BY
    t.[任务下发时间] DESC;

```
