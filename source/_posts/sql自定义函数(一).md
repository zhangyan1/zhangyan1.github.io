---
title: sql自定义函数(一)
date: 2017-04-17 00:21:02
categories: sql
tags: [sql,oracle]
keywords: function, 自定义
---

&emsp;&emsp;有时候在较为复杂的查询中,会出现较为复杂的逻辑,直接写在sql当中一来不够明了,二来会令二次修改的时候变得尤为困难不利于维护,既然是搞java的,面相对象编程自然要学会封装,所以自定义function就出现了它的意义,可以让sql便于维护一目了然,能更加方便的处理较为复杂的关联查询逻辑等等好处
<!--more--> 

&emsp;&emsp;废话不多说,让我们开始先从简单的sql自定义function说起吧,数据库中函数包含四个部分：声明、返回值、函数体和异常处理,首先来看看无参数的自定义function如下:
&emsp;&emsp;首先看一个最简单的无参数自定义函数

```
create or replace function test_one return  varchar2 --创建声明函数以及返回类型
is
rs varchar2(50);--定义varchar2类型变量
begin
	rs:='test_one';--赋值
	return rs;--返回结果
end;

```
执行`select test_one from dual`查看结果

&emsp;&emsp;再来看一个带参数相加函数
```
create or replace function test_add(
augend in number,--参数in:为只读模式,在函数中,参数的值只能被引用,不能被改变;out:为只写模式,只能被赋值,不能被引用;in out:可读可写
addend in number
) 
return  number --函数返回类型
as
rs number(18,2);--定义变量
begin
	rs:= augend + addend;
	return rs;
end;

```
执行`select test_add(1,2) from dual`查看结果

&emsp;&emsp;有些同学可能会说自定义函数就这样？那还不如直接写sql呢,好接下来我们来个稍微复杂点的
```

/*********************************************************************
名称：f_dim_getTransModel
功能描述：获取产品内转模式，1-产品组合 2-产品产品 3-非内转行内 4-非内转行外  
**********************************************************************/
create or replace function f_dim_getTransModel(
	clearNo IN varchar2
)return varchar2 is
	retval varchar2(10):='2';--变量并赋予默认值
	inner_transno  varchar2(30):='';
	hf_flag     varchar2(10):='';
	rs_number   number(18):=0;
	agent_vouchersrc atagentvouchersrctmp%rowtype;--%rowtype表示该类型为行数据类型,相当于表中的一行数据
begin
	select * into agent_vouchersrc from atagentvouchersrctmp where c_clear_no=clearNo and c_type='1';--获取一行数据并用select into赋值
	select count（*）into rs_number from (select 'X' from tb_financ_investassetno a where a.fld_assetno=agent_vouchersrc.c_serialno);
	if rs_number>=1 then --if else 分支处理逻辑
		select c_inner_transno,c_hf_flag into inner_transno,hf_flag from tb_financ_investassetno  where  fld_assetno=agent_vouchersrc.c_serialno;--直接.可以获取对应属性
	else
		retval := '4';
		return retval;
	end if;--每一个if都要对应一个end if;
	if trim(inner_transno) is null then
		if hf_flag ='1' then
			retval := '3';
		elsif hf_flag='2' then
			retval := '4';
		else 
			retval := '4';
		end if;
	else 
		select count（*）into rs_number from (select 'X' from tb_abundle_investassetno a where a.fld_assetno=inner_transno);
		if rs_number>=1 then 
			retval := '1';
		else
			retval := '2';
		end if;
	end if;
	return retval;
end;
/

```
&emsp;&emsp;上述这个函数如果用sql编写,既不好写,也容易让人看得晕头转向,但当变成函数封装起来之后,就像java程序一样你只需要关心你传入的参数和他返回给你的值就好,减轻日后维护的工作量。

&emsp;&emsp;好了今天我们先简要介绍这么多,下次介绍更为复杂的自定义函数包括与游标的结合使用,以及多层函数的调用等等。



