索引提示为优化器提供关于如何在查询处理期间选择索引的信息。这里描述的索引提示不同于8.9.3节“优化器提示”中描述的优化器提示。
索引和优化器提示可以单独使用，也可以一起使用。

索引提示仅适用于SELECT和UPDATE语句。

索引提示是在表名后面指定的。(关于在SELECT语句中指定表的一般语法，请参见13.2.9.2节“JOIN子句”。)引用单个表(包括索引提示)的语法是这样的:
	tbl_name [[AS] alias] [index_hint_list]

	index_hint_list:
		index_hint [index_hint] ...
	
	index_hint:
		USE {INDEX|KEY}
		  [FOR {JOIN|ORDER BY|GROUP BY}] ([index_list])
	  | {IGNORE|FORCE} {INDEX|KEY}
		  [FOR {JOIN|ORDER BY|GROUP BY}] (index_list)
	
	index_list:
		index_name [, index_name] ...

使用索引(index_list)提示告诉MySQL只使用其中一个已命名索引来查找表中的行。替代语法忽略索引(index_list)告诉MySQL不要使用某些特定的索引。
如果EXPLAIN显示MySQL使用了可能索引列表中的错误索引，那么这些提示很有用。

FORCE索引提示的作用类似于USE INDEX (index_list)，另外假设表扫描的开销非常大。换句话说，只有在无法使用指定索引查找表中的行时，才会使用表扫描。

每个提示都需要索引名，而不是列名。要引用主键，请使用名称primary。要查看表的索引名，请使用SHOW index语句或INFORMATION_SCHEMA。统计数据表。

index_name值不需要是完整的索引名。它可以是索引名的明确前缀。如果前缀是不明确的，就会发生错误。

例如：
	SELECT * FROM table1 USE INDEX (col1_index,col2_index)
	  WHERE col1=1 AND col2=2 AND col3=3;

	SELECT * FROM table1 IGNORE INDEX (col3_index)
	  WHERE col1=1 AND col2=2 AND col3=3;

索引提示的语法有以下特征:
	1、从语法上讲，省略index_list使用INDEX是有效的，这意味着“不使用索引”。省略index_list作为FORCE INDEX或IGNORE INDEX是一个语法错误。
	2、您可以通过在提示中添加FOR子句来指定索引提示的范围。这为优化器选择查询处理的各个阶段的执行计划提供了更细粒度的控制。
	若要仅影响MySQL决定如何查找表中的行以及如何处理连接时使用的索引，请使用FOR JOIN。若要影响用于对行进行排序或分组的索引使用，请使用ORDER BY或GROUP BY。
	3、您可以指定多个索引提示:
		SELECT * FROM t1 USE INDEX (i1) IGNORE INDEX FOR ORDER BY (i2) ORDER BY a;
	   在几个提示中命名同一个索引不是错误(即使是在同一个提示中):
	    SELECT * FROM t1 USE INDEX (i1) USE INDEX (i1,i1);
	   但是，在同一个表中混合USE INDEX and FORCE INDEX是错误的:
	    SELECT * FROM t1 USE INDEX FOR JOIN (i1) FORCE INDEX FOR JOIN (i2);
		
如果索引提示不包含 FOR子句，则提示的作用域将应用于语句的所有部分。例如，这个提示:
	IGNORE INDEX (i1)
	相当于提示的组合:
	IGNORE INDEX FOR JOIN (i1)
	IGNORE INDEX FOR ORDER BY (i1)
	IGNORE INDEX FOR GROUP BY (i1)
	
在MySQL 5.0中，没有FOR子句的提示作用域只应用于行检索。若要使服务器在没有FOR子句时使用此旧行为，请在服务器启动时启用旧系统变量。
请注意在复制设置中启用此变量。对于基于语句的二进制日志记录，对于源和副本具有不同的模式可能会导致复制错误。

在处理索引提示时，将按类型(使用、强制、忽略)和范围(对于JOIN、ORDER by、GROUP by)将它们收集到单个列表中。为
	SELECT * FROM t1
	  USE INDEX () IGNORE INDEX (i2) USE INDEX (i1) USE INDEX (i2);
    等价于:
	SELECT * FROM t1
	  USE INDEX (i1,i2) IGNORE INDEX (i2);
	  
然后索引提示按以下顺序应用于每个范围:
	{USE|FORCE} INDEX ：如果能用得上，则使用该索引。(如果用不上，则使用优化器确定的索引集。)
	IGNORE INDEX ：用于覆盖使用的结果，例如以下是等价的：
		SELECT * FROM t1 USE INDEX (i1) IGNORE INDEX (i2) USE INDEX (i2);
		SELECT * FROM t1 USE INDEX (i1);
		
