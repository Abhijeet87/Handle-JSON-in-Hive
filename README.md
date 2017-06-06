# Handle-JSON-in-Hive
Listing my experiments with JSON files in Hive

Simple JSON files can be handeled using Hives native functions, is stored in separate PDF

A SerDe is a better choice than a json function (UDF) for at least two reasons:

1.    it only has to parse each JSON record once
2.    you can define the JSON schema in the Hive table schema, making it much easier to issue queries against.



-------------------------
INPUT DATA:
First of all you have to validate your json file on http://jsonlint.com/ after that make your file as one row per line and remove the [ ]. the comma at the end of the line is mandatory.

```
>{"id": "0001","type": "donut","name": "Cake","ppu": 0.55,"batters": {"batter": [{"id": "1001","type": "Regular"},{"id": "1002","type": "Chocolate"},{"id": "1003","type": "Blueberry"},{"id": "1004","type": "Devil's Food"}]},"topping": [{"id": "5001","type": "None"},{"id": "5002","type": "Glazed"},{"id": "5005","type": "Sugar"},{"id": "5007","type": "Powdered Sugar"},{"id": "5006","type": "Chocolate with Sprinkles"},{"id": "5003","type": "Chocolate"},{"id": "5004","type": "Maple"}]}
```
----------------------
<br>FORMATTED
```html
>{
	"id": "0001",
	"type": "donut",
	"name": "Cake",
	"ppu": 0.55,
	"batters":
		{
			"batter":
			[
					{ "id": "1001", "type": "Regular" },
					{ "id": "1002", "type": "Chocolate" },
					{ "id": "1003", "type": "Blueberry" },
					{ "id": "1004", "type": "Devil's Food" }
				]
		},
	"topping":
		[
			{ "id": "5001", "type": "None" },
			{ "id": "5002", "type": "Glazed" },
			{ "id": "5005", "type": "Sugar" },
   { "id": "5007", "type": "Powdered Sugar" },
			{ "id": "5006", "type": "Chocolate with Sprinkles" },
		{ "id": "5003", "type": "Chocolate" },
			{ "id": "5004", "type": "Maple" }
		]
}
```
------------------------
<br>using rcongiu's Hive-JSON SerDe.
<br>ADD JAR /path/to/json-serde-1.1.6.jar;

<br>You can do this either at the hive prompt or put it in your $HOME/.hiverc file

<br>if you want to use hive's native serde
<br>ADD JAR /home/cloudera/Downloads/jar_files/jar_files/hive-hcatalog-core-2.0.1.jar

<br>define the Hive schema that this SerDe expects and load the Nested_data.json doc:

```sql
 CREATE EXTERNAL TABLE format.json_serde (
  id string,
  type string,
  name string,
  ppu float,		
  batters struct<batter:array<struct<id:string,type:string>>>,
  topping array< struct<id:string,type:string>>
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe' 
//ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe';(if you are using rcongiu's serde) 
stored as textfile;
LOAD DATA INPATH '/user/cloudera/STG/JSON/Nested_data_1line.json' OVERWRITE INTO TABLE format.json_serde;
```
<br>First let's query something from each document section. Since we know there are four id,type in the batter array we can <br>reference them both directly

```sql
  SELECT id, ppu,
       batters.batter[0].id as batter0id,
       batters.batter[1].id as batter1id,
       topping[0].id as topping0id,
       topping[1].id as topping1id
  FROM format.json_serde;
```
<br>RESULT
<br>OK

|id |	  ppu|	  batter0id|	batter1id|	topping0id|	topping1id |
|---|:-----|:----------:|:-------:|:--------:|----------:|
|0001|	0.55|	1001        |		  1002|		  5001|		    5002|


<br>If you dont know the number of elements

```sql
SELECT id, type, ppu, batters.batter.id, batters.batter.type, topping.type
    FROM format.json_serde;
```
<br>RESULT

id|	  type|	ppu|	  id	|			                type|							type|
---|----|----|--------|-------------------|------------
0001 |	donut |	0.55	| ["1001","1002","1003","1004"]	| ["Regular","Chocolate","Blueberry","Devil's Food"] |["None","Glazed","Sugar","Powdered Sugar","Chocolate with Sprinkles","Chocolate","Maple"]

<br>Time taken: 0.476 seconds, Fetched: 1 row(s)




<br>Troubleshoot: to see if Serde is reading the data proprly or not, you may turn 
<br>ALTER TABLE format.json_serde  SET SERDEPROPERTIES ( "ignore.malformed.json" = "true"); 
 



