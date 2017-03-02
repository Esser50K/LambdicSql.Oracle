﻿LambdicSql for Oracle_β 0.1.0
======================

## Features ...
LambdicSql is ultimate sql builder.<br>
Generate sql text and parameters from lambda. <br>
We will make it possible to represent as many Oracle clause and functions as possible in C #. <br>
<img src="https://github.com/Codeer-Software/LambdicSql/blob/master/lambdicSqlImage.png" width="400">
## Getting Started
LambdicSql from NuGet

    PM> Install-Package LambdicSql.Oracle

https://www.nuget.org/packages/LambdicSql.Oracle/<br>

Supported pratforms are
- .NETFramework 3.5~
- PCL
- .NETStandard 1.2~

## Featuring Dapper
Generate sql text and parameters by LambdicSql.<br>
And execut and map to object, recommend using dapper.

    PM> Install-Package Dapper

First you need to put the initialization code.
```csharp
//full .net
DapperAdapter.Assembly = typeof(Dapper.SqlMapper).Assembly;

//.net standard
DapperAdapter.Assembly = typeof(Dapper.SqlMapper).GetTypeInfo().Assembly;
```

## Quick Start.
Standard code.
```csharp
using System;
using System.Linq;
using LambdicSql;
using LambdicSql.Oracle;
using static LambdicSql.Oracle.Symbol;
using Oracle.ManagedDataAccess.Client;
using LambdicSql.feat.Dapper;
using System.Data;

namespace Sample
{
    //tables.
    public class Staff
    {
        public int id { get; set; }
        public string name { get; set; }
    }

    public class Remuneration
    {
        public int id { get; set; }
        public int staff_id { get; set; }
        public DateTime payment_date { get; set; }
        public decimal money { get; set; }
    }

    public class DB
    {
        public Staff tbl_staff { get; set; }
        public Remuneration tbl_remuneration { get; set; }
    }

    public class SelectData
    {
        public string Name { get; set; }
        public DateTime PaymentDate { get; set; }
        public decimal Money { get; set; }
        public decimal Total { get; set; }
        public string Type { get; set; }
        public int Rank { get; set; }
    }

    class Program
    {
        static void Main(string[] args)
        {
            //initialize dapper.
            DapperAdapter.Assembly = typeof(Dapper.SqlMapper).Assembly;

            //test.
            using (var cnn = new OracleConnection("your connection string")) Sample(cnn);
        }

        static void Sample(IDbConnection cnn)
        {
            //can use variable.
            var min = 1000;
            var caseMiddle = 3000;
            string middleTypeName = "Middle";

            //make sql.
            var sql = Db<DB>.Sql(db =>

                //--------------lambda.----------------------------------------------------
                Select(new SelectData
                {
                    Name = db.tbl_staff.name,
                    PaymentDate = db.tbl_remuneration.payment_date,
                    Money = db.tbl_remuneration.money,

                    //sub query
                    Total = Select(Sum(db.tbl_remuneration.money)).From(db.tbl_remuneration),

                    //case
                    Type = Case().
                                When(db.tbl_remuneration.money < 2000).Then("Cheap").
                                When(db.tbl_remuneration.money < caseMiddle).Then(middleTypeName).
                                Else("High").
                            End(),

                    //window
                    Rank = Rank().
                           Over(PartitionBy(db.tbl_remuneration.payment_date),
                                OrderBy(Asc(db.tbl_remuneration.money)))
                }).
                From(db.tbl_remuneration).
                    Join(db.tbl_staff, db.tbl_staff.id == db.tbl_remuneration.staff_id).
                Where(min < db.tbl_remuneration.money && db.tbl_remuneration.money < 4000).
                OrderBy(Asc(db.tbl_remuneration.money), Desc(db.tbl_staff.name))
                //--------------------------------------------------------------------------

                );

            //to string and params.
            var info = sql.Build(cnn.GetType());

            //print.
            Console.WriteLine(info.Text);
            Console.WriteLine("\r\nParams");
            foreach (var e in info.GetParams()) Console.WriteLine($"{e.Key} = {e.Value.Value}");

            //execute query by dapper or sql-net-pcl.
            var datas = cnn.Query(sql).ToList();
        }
    }
}
```
<img src="https://github.com/Codeer-Software/LambdicSql/blob/master/SummaryCode.png"><br>
## Samples
Look for how to use from the tests.<br>
https://github.com/Codeer-Software/LambdicSql.Oracle/tree/master/Project/Test45<br>

## Sub query.
```csharp
public void TestSubQueryAtSelect()
{
    var sql = Db<DB>.Sql(db =>
        Select(new SelectedData
        {
            Name = db.tbl_staff.name,
            Total = Select(Sum(db.tbl_remuneration.money)).
                        From(db.tbl_remuneration)
        }).
        From(db.tbl_staff));
        
    //Subqueries can also be written separately.
    //Here is the same SQL as above.
    var sub = Db<DB>.Sql(db => 
        Select(Sum(db.tbl_remuneration.money)).
        From(db.tbl_remuneration)
        );

    sql = Db<DB>.Sql(db =>
        Select(new SelectedData
        {
            Name = db.tbl_staff.name,
            Total = sub
        }).
        From(db.tbl_staff));
}
```
```sql
SELECT
        tbl_staff.name AS Name,
        (SELECT
                SUM(tbl_remuneration.money)
        FROM tbl_remuneration) AS Total
FROM tbl_staff
```
```csharp
public void TestSubQueryAtFrom()
{
    //For the where clause, you need to write the subqueries separately.
    var sub = Db<DB>.Sql(db => 
        Select(new 
        {
            name_sub = db.tbl_staff.name,
            PaymentDate = db.tbl_remuneration.payment_date,
            Money = db.tbl_remuneration.money,
        }).
        From(db.tbl_remuneration).
            Join(db.tbl_staff, db.tbl_remuneration.staff_id == db.tbl_staff.id).
        Where(3000 < db.tbl_remuneration.money && db.tbl_remuneration.money < 4000));
    
    var sql = Db<DB>.Sql(db => 
        Select(new SelectData
        {
            Name = sub.Body.name_sub
        }).
        From(sub));
}
```
```sql
SELECT
	subQuery.name_sub AS Name
FROM 
	(SELECT
		tbl_staff.name AS name_sub,
		tbl_remuneration.payment_date AS PaymentDate,
		tbl_remuneration.money AS Money
	FROM tbl_remuneration
		JOIN tbl_staff ON (tbl_remuneration.staff_id) = (tbl_staff.id)
	WHERE ((@p_2) < (tbl_remuneration.money)) AND ((tbl_remuneration.money) < (@p_3))) AS subQuery
```
## Combining queries.
It can be combined parts.
Query, Sub query, Expression.
You can write DRY code.
```csharp
public void TestQueryConcat()
{
    //make sql.
    var select = Db<DB>.Sql(db =>
        Select(new SelectData1
        {
            Name = db.tbl_staff.name,
            PaymentDate = db.tbl_remuneration.payment_date,
            Money = db.tbl_remuneration.money,
        }));

    var from = Db<DB>.Sql(db =>
         From(db.tbl_remuneration).
        Join(db.tbl_staff, db.tbl_remuneration.staff_id == db.tbl_staff.id));

    var where = Db<DB>.Sql(db =>
        Where(3000 < db.tbl_remuneration.money && db.tbl_remuneration.money < 4000));

    var orderby = Db<DB>.Sql(db =>
         OrderBy(Asc(db.tbl_staff.name)));

    var sql = select + from + where + orderby;

    //to string and params.
    var info = sql.Build(_connection.GetType());
    Debug.Print(info.Text);

    //dapper
    var datas = _connection.Query(sql).ToList();
}
```
```csharp
public void TestClauseConcat()
{
    //make sql.
    var where = Db<DB>.Sql(db =>
        Where(3000 < db.tbl_remuneration.money && db.tbl_remuneration.money < 4000));

    //make sql.
    var sql = Db<DB>.Sql(db =>
        Select(new SelectData
        {
            Name = db.tbl_staff.name,
            PaymentDate = db.tbl_remuneration.payment_date,
            Money = db.tbl_remuneration.money,
        }) + 
        
        From(db.tbl_remuneration) +
            Join(db.tbl_staff, db.tbl_staff.id == db.tbl_remuneration.staff_id) +

        where +    

        OrderBy(Asc(db.tbl_remuneration.money))
        );

    //to string and params.
    var info = sql.Build(_connection.GetType());
    Debug.Print(info.Text);

    //dapper
    var datas = _connection.Query(sql).ToList();
}
```
```sql
SELECT
	tbl_staff.name AS Name,
	tbl_remuneration.payment_date AS PaymentDate,
	tbl_remuneration.money AS Money
FROM tbl_remuneration
	JOIN tbl_staff ON (tbl_remuneration.staff_id) = (tbl_staff.id)
WHERE ((@min) < (tbl_remuneration.money)) AND ((tbl_remuneration.money) < (@p_1))
ORDER BY
	tbl_staff.name ASC
```
Expressions.
```csharp
public void TestSqlExpression()
{
    //make sql.
    var expMoneyAdd = Db<DB>.Sql(db => db.tbl_remuneration.money + 100);
    var expWhereMin = Db<DB>.Sql(db => 3000 < db.tbl_remuneration.money);
    var expWhereMax = Db<DB>.Sql(db => db.tbl_remuneration.money < 4000);

    var sql = Db<DB>.Sql(db =>
       Select(new SelectData1
       {
           Name = db.tbl_staff.name,
           PaymentDate = db.tbl_remuneration.payment_date,
           Money = expMoneyAdd,
       }).
       From(db.tbl_remuneration).
           Join(db.tbl_staff, db.tbl_remuneration.staff_id == db.tbl_staff.id).
       Where(expWhereMin && expWhereMax).
       OrderBy(Asc(db.tbl_staff.name)));

    //to string and params.
    var info = sql.Build(_connection.GetType());
    Debug.Print(info.Text);

    //dapper
    var datas = _connection.Query(sql).ToList();
}
```
```sql
SELECT
	tbl_staff.name AS Name,
	tbl_remuneration.payment_date AS PaymentDate,
	(tbl_remuneration.money) + (@p_0) AS Money
FROM tbl_remuneration
	JOIN tbl_staff ON (tbl_remuneration.staff_id) = (tbl_staff.id)
WHERE ((@p_1) < (tbl_remuneration.money)) AND ((tbl_remuneration.money) < (@p_2))
ORDER BY
	tbl_staff.name ASC
```

## Condition utility
```csharp
public void TestCondition(bool minCondition, bool maxCondition)
{
    //Condition is written only when the first argument is valid.
    var exp = Db<DB>.Sql(db =>
        new Condition(minCondition, 3000 < db.tbl_remuneration.money) &&
        new Condition(maxCondition, db.tbl_remuneration.money < 4000));

    var query = Db<DB>.Sql(db =>
        Select(Asterisk()).
        From(db.tbl_remuneration).
        Where(exp)
    );
}
```
min max enable.
```sql
SELECT *
FROM tbl_remuneration
WHERE ((@p_0) < (tbl_remuneration.money)) AND ((tbl_remuneration.money) < (@p_1))
```
min, max disable, vanish where clause.
```sql
SELECT *
FROM tbl_remuneration
```
## Support for combination of the text.
You can insert text to expression.
And insert expression to text.
```csharp
//You can use text.
public void TestFormatText()
{
    //make sql.
    var sql = Db<DB>.Sql(db =>
        Select(new SelectedData
        {
            name = db.tbl_staff.name,
            payment_date = db.tbl_remuneration.payment_date,
            money = (decimal)"{0} + 1000".ToSql(db.tbl_remuneration.money),
        }).
        From(db.tbl_remuneration).
            Join(db.tbl_staff, db.tbl_remuneration.staff_id == db.tbl_staff.id).
        Where(3000 < db.tbl_remuneration.money && db.tbl_remuneration.money < 4000));

    //to string and params.
    var info = sql.Build(_connection.GetType());
    Debug.Print(info.SqlText);

    //if you installed dapper, use this extension.
    var datas = _connection.Query(sql).ToList();
}
```
```sql
SELECT
	tbl_staff.name AS name,
	tbl_remuneration.payment_date AS payment_date,
	tbl_remuneration.money + 1000 AS money
FROM tbl_remuneration
	JOIN tbl_staff ON (tbl_remuneration.staff_id) = (tbl_staff.id)
WHERE ((@p_0) < (tbl_remuneration.money)) AND ((tbl_remuneration.money) < (@p_1))
```
## 2 way sql.
Do you know 2 way sql?
It's executable sql text.
And change by condition and keyword.
```csharp
//2 way sql
public void TestFormat2WaySql()
{
    TestFormat2WaySql(true, true, 1000);
}
public void TestFormat2WaySql(bool minCondition, bool maxCondition, decimal bonus)
{
    //make sql.
    //replace /*no*/---/**/.
    var text = @"
SELECT
tbl_staff.name AS name,
tbl_remuneration.payment_date AS payment_date,
tbl_remuneration.money + /*0*/1000/**/ AS money
FROM tbl_remuneration 
JOIN tbl_staff ON tbl_staff.id = tbl_remuneration.staff_id
/*1*/WHERE tbl_remuneration.money = 100/**/";
    
    var sql = Db<DB>.Sql(db => text.TwoWaySql(
        bonus,
        Where(
            new Condition(minCondition, 3000 < db.tbl_remuneration.money) &&
            new Condition(maxCondition, db.tbl_remuneration.money < 4000))
        ));
    var info = sql.Build(_connection.GetType());
    Debug.Print(info.SqlText);

    //if you installed dapper, use this extension.
    var datas = _connection.Query<SelectedData>(sql).ToList();
}
```
```sql
SELECT
	tbl_staff.name AS name,
    tbl_remuneration.payment_date AS payment_date,
	tbl_remuneration.money + @bonus AS money
FROM tbl_remuneration 
    JOIN tbl_staff ON tbl_staff.id = tbl_remuneration.staff_id
WHERE ((@p_0) < (tbl_remuneration.money)) AND ((tbl_remuneration.money) < (@p_1))
```
