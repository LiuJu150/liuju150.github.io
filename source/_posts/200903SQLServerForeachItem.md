---
layout: post
title: 'SqlServer遍历数据'
subtitle: '批量修改数据库表名前缀为大写'
header-img: "img/win.jpg"
tags:
  - 随笔
  - SqlServer
date: 2020-09-03 08:51:31
---

## 前言

数据库原有表名为：bas_LocTemp
要修改后表名为：BAS_LocTemp

## 代码

```sql
--给要遍历的数据增加行号，如果有ID可以用ID字段，并保存到临时表temp
select ROW_NUMBER() over(order by name) as RowId,name as TableName into #temp from sys.tables where type='U'
declare @CurRowId int --当前索引
select @CurRowId=MIN(RowId) from #temp  --第一次先找到最小的索引号(行号)
print @CurRowId
while(@CurRowId is not null) --遍历数据,直到没有数据为止
begin
	print @CurRowId
	declare @TableName varchar(128)
	select @TableName=TableName from #temp where RowId=@CurRowId --根据当前索引找到要修改的表名
	print @TableName
	
	declare @NewTableName varchar(128)
  --根据老的表名，生成新的表名
	select @NewTableName=UPPER(SUBSTRING(@TableName,0,CHARINDEX('_',@TableName,0)))+SUBSTRING(@TableName,CHARINDEX('_',@TableName,0),LEN(@TableName))
	print @NewTableName
	exec sp_rename @TableName,@NewTableName --使用系统存储过程修改表名
	select @CurRowId=MIN(RowId) from #temp where RowId>@CurRowId --查找下一条要修改的数据索引
end
drop table #temp --删除临时表temp
```

## 修改表字段类型

```sql
select 
	ROW_NUMBER() over(order by t.name,c.name) as RowId,
	t.name as tName,c.name as cName,c.is_nullable as cNull
into #temp
from sys.columns c
inner join sys.tables t on t.object_id=c.object_id
where c.user_type_id=106 and t.type='U' and c.precision=38 and c.scale=30

declare @CurRowId int --当前索引
select @CurRowId=MIN(RowId) from #temp
print @CurRowId
while(@CurRowId is not null)
begin
	print @CurRowId
	declare @TableName varchar(128)
	declare @ColumnName varchar(128)
	declare @Nullable bit
	select @TableName=tName,@ColumnName=cName,@Nullable=cNull from #temp where RowId=@CurRowId

	declare @sql varchar(1000)

	if(@Nullable=0)
	begin
		set @sql='ALTER TABLE '+@TableName+' ALTER COLUMN '+@ColumnName+' decimal(22,5) NOT NULL'
	end
	else
	begin
		set @sql='ALTER TABLE '+@TableName+' ALTER COLUMN '+@ColumnName+' decimal(22,5) NULL'
	end
	print @sql
	begin try
		exec(@sql)
	end try
	begin catch
	 print @sql+' 执行报错'
	 print ERROR_MESSAGE()
	end catch

	select @CurRowId=MIN(RowId) from #temp where RowId>@CurRowId
end
drop table #temp
```

## 结语

对于SqlServer遍历数据，有很多方法。只文只记录一个通用的方式
如果只用来遍历表，SqlServer也自带了系统存储过程，这里就记录了
