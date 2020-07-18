# 复杂SQL语句

**1、SQL统计某字段的出现次数**

比如统计某个表中，姓名出现的次数：

```sql
select name,count(*) from biao group by name having count(*) > 2
```

关键是用分组：group by，且经常和聚合函数一起使用

比如：统计用户表中的匿名字段的出现次数

```sql
SELECT  a.user_anon_name as anon_name  , COUNT(a.user_anon_name) as num FROM xm_user a GROUP BY a.user_anon_name
```

标准写法：

```sql
select 类别, sum(数量) as 数量之和 from A group by 类别 having sum(数量) > 18
```

 **2、SQL中如何用一个表的列更新另一个表的列**

```sql
--update T2 set 要更新的字段 = T1.对应的字段　可以输入多个用逗号分开 
from T1 where T2.ID = T1.ID
```

**3、SQL 多字段排序**

```sql
select * from (select * from tablename order by last_time desc) as t order by t.id desc
```

**4 、列出所有员工的年工资，按年薪从低到高排序**

```sql
SELECT (SAL+NVL(COMM,0))*12 INCOME FROM EMP ORDER BY INCOME;
```

**5 、列出薪金比“ SMITH ”多的所有员工**(获取SMITH 的工资)

```sql
SELECT * FROM EMP WHERE SAL >(SELECT SAL FROM EMP WHERE ENAME = 'SMITH');
```

**6 、列出所有员工的姓名及其直接上级的姓名**

```sql
SELECT E1.ENAME,E2.ENAME FROM EMP E1 LEFT JOIN EMP E2 ON E1.MGR = E2.EMPNO;
```

**7 、列出受雇日期早于其直接上级的所有员工**

```sql
SELECT E1.ENAME,E1.HIREDATE,E2.ENAME,E2.HIREDATE FROM EMP E1 LEFT JOIN EMP E2 
ON E1.MGR = E2.EMPNO WHERE E1.HIREDATE <E2.HIREDATE;
```

**8 、列出部门名称和这些部门的员工信息，包括那些没有员工的部门**

```sql
SELECT D.DNAME,E.EMPNO,E.ENAME,DEPTNO FROM EMP E RIGHT JOIN DEPT D USING(DEPTNO);
```

**9 、列出所有job 为“ CLERK ”（办事员）的姓名及其部门名称**

```sql
SELECT D.DNAME,T.ENAME FROM (SELECT DEPTNO,ENAME FROM EMP WHERE JOB ='CLERK') T,DEPT D
WHERE T.DEPTNO = D.DEPTNO;
```

**10 、列出最低工资大于1500 的各种工作**

```sql
SELECT JOB FROM EMP GROUP BY JOB HAVING MIN(SAL)>1500;
```

**11 、列出在部门“ SALES ”（销售部）工作的员工的姓名，假定不知道销售部的部门编号**

```sql
SELECT ENAME,DEPTNO FROM EMP
WHERE DEPTNO = (SELECT DEPTNO FROM DEPT WHERE DNAME = 'SALES');
```

**12 、列出工资高于公司平均工资的所有员工**
获取公司的平均工资

```sql
SELECT * FROM EMP WHERE SAL >(SELECT ROUND(AVG(SAL)) AVG_SAL FROM EMP);
```

**13 、列出与“ SCOTT ”从事相同工作的所有员工**，获取SCOTT的工作

```sql
SELECT * FROM EMP WHERE JOB = (SELECT JOB FROM EMP WHERE ENAME = 'SCOTT') AND ENAME <> 'SCOTT';
```



**14 、列出工资高于在部门 30 工作的所有员工的工资的员工姓名和工资**

```sql
SELECT ENAME,SAL FROM EMP WHERE SAL >ALL(SELECT SAL FROM EMP WHERE DEPTNO = 30);
```

**15 、列出在每个部门工作的员工数量、平均工资和平均服务期限（年）**

```sql
SELECT COUNT(*),ROUND(AVG(SAL)) AVG_SAL,ROUND(AVG(TO_CHAR(SYSDATE,'YYYY')-TO_CHAR(HIREDATE,'YYYY'))) AVG_YEAR FROM EMP GROUP BY DEPTNO;
```



**16 、列出所有员工的姓名、部门名称和工资**

```sql
SELECT E.ENAME,D.DNAME,E.SAL FROM EMP E NATURAL JOIN DEPT D;
```

**17 、列出从事同一种工作但属于不同部门的员工的一种组合**

```sql
--[自连接]E1 和 E2 都当成员工表
SELECT E1.ENAME,E1.JOB,E1.DEPTNO,E2.ENAME,E2.JOB,E2.DEPTNO FROM EMP E1,EMP E2
WHERE E1.JOB = E2.JOB AND E1.DEPTNO <> E2.DEPTNO AND E1.ENAME<E2.ENAME;
```


