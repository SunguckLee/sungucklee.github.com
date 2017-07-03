
Intro
===============
MySQL supports geometry data type and spatial index with R-Tree index algorithm.
But this spatial index based on R-Tree index algorithm may be sufficient or not based on your service requirements.

If your service need to search categorized-poi data (like below example), then you may want to make compound index with category column.

```sql
SELECT * FROM poi 
WHERE type='cafe' 
  AND ST_Within(location, ST_GeomFromText('POLYGON((127.041697 37.551675, 127.053327 37.551675, 127.053327 37.542488, 127.041697 37.542488, 127.041697 37.551675))'));
```

But MySQL spatial index (based on R-Tree index algorithm) does not allow combining geometry type and regular column.

```sql
CREATE TABLE poi (
  id BIGINT NOT NULL AUTO_INCREMENT,
  name VARCHAR(30) NOT NULL,
  type ENUM('cafe', 'restaurant', 'bakery', 'bar'),
  location GEOMETRY NOT NULL, 
  PRIMARY KEY (id),
  SPATIAL INDEX sx_location(type, location)
) ENGINE=InnoDB;

>> ERROR 1070 (42000): Too many key parts specified; max 1 parts allowed
```

So MySQL optimizer can not utilize single index for both type and location column condition. Usually MySQL optimizer will choose spatial index for execution plan, but in this case MySQL has to read a lot of useless records (to filter out type<>'cafe').
Even worse thing is that MySQL has to access(random access) full record to check "type='cafe'" condition because spatial index does not have type column data.


Why S2 geometry
===============
This is why I added location(POINT) search feature using s2 geometry to MySQL.
S2 geometry library prvoides a lot of functionality to implement goemetry search including below two main function.
- Convert vector data to scalar value using hilbert curve.
- Generate target cell(s2 cell id) to cover interesting area.

S2 geometry search plugin for MySQL use this two main function of S2 geometry library and S2 geometry search plugin is based on MySQL 5.7 Query rewrite plugin. Actually S2 geometry search plugin is query rewrite plugin itself. And this patch support a few function to convert longitude and latitude pair to s2 cell id and vice-versa.

MySQL spatial index
-------------------
First let's check how we can implement poi search with mysql spatial index.

```sql
-- // Create poi table with spatial index on location gemetry type column
CREATE TABLE poi (
  id BIGINT NOT NULL AUTO_INCREMENT,
  name VARCHAR(30) NOT NULL,
  type ENUM('cafe', 'restaurant', 'bakery', 'bar'),
  location GEOMETRY NOT NULL, 
  PRIMARY KEY (id),
  SPATIAL INDEX sx_location(location)
) ENGINE=InnoDB;

-- // Insert a few sample data
INSERT INTO poi(id, location, type, name) VALUES (NULL, POINT(127.048764, 37.547287), "bakery",     "T-jure");
INSERT INTO poi(id, location, type, name) VALUES (NULL, POINT(127.048372, 37.547516), "bakery",     "P-baguette");
INSERT INTO poi(id, location, type, name) VALUES (NULL, POINT(127.047235, 37.546623), "restaurant", "K-Noodle");
INSERT INTO poi(id, location, type, name) VALUES (NULL, POINT(127.044773, 37.546989), "restaurant", "H-Burger");
INSERT INTO poi(id, location, type, name) VALUES (NULL, POINT(127.046296, 37.546325), "bar",        "Chicken");
INSERT INTO poi(id, location, type, name) VALUES (NULL, POINT(127.044891, 37.545892), "cafe",       "P-Dabang");
INSERT INTO poi(id, location, type, name) VALUES (NULL, POINT(127.044854, 37.546618), "bakery",     "Mildo");
INSERT INTO poi(id, location, type, name) VALUES (NULL, POINT(127.045444, 37.546499), "bar",        "B-Beer");

-- // Check inserted data
mysql> select id, name, type, ST_AsText(location) from poi;
+----+------------+------------+-----------------------------+
| id | name       | type       | AsText(location)            |
+----+------------+------------+-----------------------------+
|  1 | T-jure     | bakery     | POINT(127.048764 37.547287) |
|  2 | P-baguette | bakery     | POINT(127.048372 37.547516) |
|  3 | K-Noodle   | restaurant | POINT(127.047235 37.546623) |
|  4 | H-Burger   | restaurant | POINT(127.044773 37.546989) |
|  5 | Chicken    | bar        | POINT(127.046296 37.546325) |
|  6 | P-Dabang   | cafe       | POINT(127.044891 37.545892) |
|  7 | Mildo      | bakery     | POINT(127.044854 37.546618) |
|  8 | B-Beer     | bar        | POINT(127.045444 37.546499) |
+----+------------+------------+-----------------------------+

-- // Make query to search poi in the specific boudning box with rectangle polygon
mysql> SELECT id, name, type, ST_AsText(location) 
FROM poi 
WHERE type='cafe' 
  AND ST_Within(location, ST_GeomFromText('POLYGON((127.041697 37.551675, 127.053327 37.551675, 127.053327 37.542488, 127.041697 37.542488, 127.041697 37.551675))'));
+----+----------+------+-----------------------------+
| id | name     | type | ST_AsText(location)         |
+----+----------+------+-----------------------------+
|  6 | P-Dabang | cafe | POINT(127.044891 37.545892) |
+----+----------+------+-----------------------------+
```

But MySQL has to find all poi data within given rectangle area regardless of type column value if optimizer choose spatial index. The case using index on type column(if exist) is also non-optimized because MySQL has to find all poi data regardless of matching rectangle area.


S2 geometry index
-------------------
Now let's check the way to implement same search feature using S2 geometry search plugin.

```sql
-- // Create table
CREATE TABLE poi_s2 (
  id BIGINT NOT NULL AUTO_INCREMENT,
  name VARCHAR(30) NOT NULL,
  type ENUM('cafe', 'restaurant', 'bakery', 'bar'),
  s2location BIGINT UNSIGNED NOT NULL, 
  PRIMARY KEY (id),
  /* Compound index with (type + s2location), s2location column is just bigint type so we can make compound index */
  INDEX ix_type_location(type, s2location)
) ENGINE=InnoDB;

-- // Insert sample data
INSERT INTO poi_s2(id, s2location, type, name) VALUES (NULL, S2CELL(37.547287, 127.048764), "bakery",     "T-jure");
INSERT INTO poi_s2(id, s2location, type, name) VALUES (NULL, S2CELL(37.547516, 127.048372), "bakery",     "P-baguette");
INSERT INTO poi_s2(id, s2location, type, name) VALUES (NULL, S2CELL(37.546623, 127.047235), "restaurant", "K-Noodle");
INSERT INTO poi_s2(id, s2location, type, name) VALUES (NULL, S2CELL(37.546989, 127.044773), "restaurant", "H-Burger");
INSERT INTO poi_s2(id, s2location, type, name) VALUES (NULL, S2CELL(37.546325, 127.046296), "bar",        "Chicken");
INSERT INTO poi_s2(id, s2location, type, name) VALUES (NULL, S2CELL(37.545892, 127.044891), "cafe",       "P-Dabang");
INSERT INTO poi_s2(id, s2location, type, name) VALUES (NULL, S2CELL(37.546618, 127.044854), "bakery",     "Mildo");
INSERT INTO poi_s2(id, s2location, type, name) VALUES (NULL, S2CELL(37.546499, 127.045444), "bar",        "B-Beer");

-- // Check inserted records
-- // S2Longitude and S2Latitude is UDF to reverse-generate latitude and longitude from s2 cell id
mysql> SELECT id, name, type, S2Longitude(s2location) as location_lon, S2Latitude(s2location) as location_lat, s2location FROM poi_s2;
+----+------------+------------+--------------------+--------------------+---------------------+
| id | name       | type       | location_lon       | location_lat       | s2location          |
+----+------------+------------+--------------------+--------------------+---------------------+
|  1 | T-jure     | bakery     | 127.04876396334656 |  37.54728698238406 | 3854136361145573953 |
|  2 | P-baguette | bakery     | 127.04837196406127 |  37.54751598392203 | 3854136361037750147 |
|  3 | K-Noodle   | restaurant | 127.04723501496323 |  37.54662298941224 | 3854136360020792807 |
|  4 | H-Burger   | restaurant | 127.04477303829745 |  37.54698896586247 | 3854136382464207171 |
|  5 | Chicken    | bar        |  127.0462959785528 |   37.5463249973006 | 3854136359654373161 |
|  6 | P-Dabang   | cafe       | 127.04489096350927 | 37.545892027418084 | 3854136371285358265 |
|  7 | Mildo      | bakery     | 127.04485403000865 |   37.5466179990086 | 3854136371093275229 |
|  8 | B-Beer     | bar        | 127.04544398106482 |  37.54649901104369 | 3854136359591207909 |
+----+------------+------------+--------------------+--------------------+---------------------+`

-- // Query to find poi which is located in the give radius circle with type='cafe'
SELECT id, name, type, S2Longitude(s2location) as location_lon, S2Latitude(s2location) as location_lat, s2location
FROM poi_s2
WHERE type='cafe' 
  AND S2WITHIN(s2location, 37.547273, 127.047171, 475)
+----+----------+------+--------------------+--------------------+---------------------+
| id | name     | type | location_lon       | location_lat       | s2location          |
+----+----------+------+--------------------+--------------------+---------------------+
|  6 | P-Dabang | cafe | 127.04489096350927 | 37.545892027418084 | 3854136371285358265 |
+----+----------+------+--------------------+--------------------+---------------------+
1 row in set, 1 warning (0.01 sec)
```

In above example, s2location column is just bigint(unsigned) column so we can make compound index using type and s2location column. And S2 geometry plugin will rewrite S2WITHIN part as normal query condition which can be parsed by MySQL optimizer. MySQL will show warning message when S2 geometry query rewrite plugin rewrites user query and you can check how query is rewritten with "SHOW WARNINGS" command.

How query is rewritten
-----------------------
```sql
mysql> SELECT * FROM poi_s2 WHERE type='cafe' AND S2WITHIN(s2location, 37.547273, 127.047171, 475, 10);
+----+----------+------+---------------------+
| id | name     | type | s2location          |
+----+----------+------+---------------------+
|  6 | P-Dabang | cafe | 3854136371285358265 |
+----+----------+------+---------------------+
1 row in set, 1 warning (0.00 sec)

mysql> SHOW WARNINGS;
Note: Query 'SELECT * FROM poi_s2 WHERE type='cafe' AND S2WITHIN(s2location, 37.547273, 127.047171, 475, 10)' rewritten to 'SELECT * FROM poi_s2 WHERE type='cafe' AND  (s2location BETWEEN 3854136322323120129 AND 3854136322457337855 OR s2location BETWEEN 3854136345274351617 AND 3854136388224024575 OR s2location BETWEEN 3854136396780404737 AND 3854136401108926463 OR s2location BETWEEN 3854136512778076161 AND 3854136514925559807) s2loc' by a query rewrite plugin
```

As shown above, "S2WITHIN(s2location, 37.547273, 127.047171, 475)" part is written to "(s2location BETWEEN 3854136322323120129 AND 3854136322457337855 OR s2location BETWEEN 3854136349569318913 AND 3854136364601704447 ...". And MySQL optimizer will prepare execution plan with this rewritten query not original user query. S2 geometry query rewrite plugin is kind of "before parser" type. Because of this "before parser" type, we can't use prepared-statement with this S2 geometry query rewrite plugin.

Filter-out mismatched poi
-------------------------
And one more thing is that we can SELECT more accurate result with fast covering index(of query execution plan).
S2 geometry also find poi data using several rectangles(diamond shape box), So the result of "S2WITHIN" search also contains poi data not contained circle area of given radius. But using S2Distance UDF, we can eliminate poi not contained in exact circle area. And this filtering task can be processed as "covering index" without accessing poi record data(with small cost). 

```sql
mysql> SELECT id, name, type, S2LONGITUDE(s2location) as location_lon, S2LATITUDE(s2location) as location_lat, s2location
FROM poi_s2
WHERE type='cafe' 
  AND S2WITHIN(s2location, 37.547273, 127.047171, 475)
  AND S2DISTANCE(s2location, 37.547273, 127.047171) <= 475;

+...+------+--------------------+---------+-------+------+----------+--------------------------+
|...| type | key                | key_len | ref   | rows | filtered | Extra                    |
+...+------+--------------------+---------+-------+------+----------+--------------------------+
|...| ref  | ix_type_s2location | 2       | const |    7 |    79.03 | Using where; Using index |
+...+------+--------------------+---------+-------+------+----------+--------------------------+
```

UDF
================
- S2CELL : Generate S2 cell id from latitude and longitude pair
```sql
mysql> SELECT S2CELL(35.198362, 129.053922) as busan;
+---------------------+
| busan               |
+---------------------+
| 3848489404846389721 |
+---------------------+

mysql> SELECT S2CELL(37.532600, 127.024612) as seoul;
+---------------------+
| seoul               |
+---------------------+
| 3854135263929674215 |
+---------------------+
```

- S2LATITUDE, S2LONGITUDE : Reverse generate latitude and longitude from S2 cell id
```sql
mysql> SELECT S2LATITUDE(s2cell(37.532600, 127.024612)) as latitude, S2LONGITUDE(s2cell(37.532600, 127.024612)) as longitude;
+-------------------+-------------------+
| latitude          | longitude         |
+-------------------+-------------------+
| 37.53260002139435 | 127.0246119770569 |
+-------------------+-------------------+
```

- S2DISTANCE : Calculate distance between S2 cell and (latitude, longitude)
```sql
mysql> SELECT S2DISTANCE(s2cell(37.532600, 127.024612), 35.198362, 129.053922) as distance_meters_from_seoul_to_busan;
+-------------------------------------+
| distance_meters_from_seoul_to_busan |
+-------------------------------------+
|                       316815.053085 |
+-------------------------------------+
```

How to install
================
```sql
-- // Install UDF
mysql> CREATE FUNCTION S2Cell RETURNS INTEGER SONAME 's2_udf.so';
mysql> CREATE FUNCTION S2Distance RETURNS REAL SONAME 's2_udf.so';
mysql> CREATE FUNCTION S2Latitude RETURNS REAL SONAME 's2_udf.so';
mysql> CREATE FUNCTION S2Longitude RETURNS REAL SONAME 's2_udf.so';

-- // Install S2 query rewrite plugin
mysql> INSTALL PLUGIN s2_geometry SONAME 's2_query_rewriter.so';


-- // Uninstall UDF
mysql> DROP FUNCTION S2Cell;
mysql> DROP FUNCTION S2Distance;
mysql> DROP FUNCTION S2Latitude;
mysql> DROP FUNCTION S2Longitude;
```


ERROR
================
Still there's no proper way to report error of Query rewrite plugin, so S2 geometry query rewrite plugin may report error using query result. This is not a good way to report error becuase the shape of query result is changed. But still I can not find better way to report error.

S2 geometry query rewrite plugin generate several errors if arguments are not valid.

- Radius must be less than 500 Km
```sql
mysql> SELECT * FROM poi_s2 WHERE type='cafe' AND S2WITHIN(s2location, 37.547273, 127.047171, 4750000);
+-------+--------------------+
| errno | errmsg             |
+-------+--------------------+
|     9 | ERR_INVALID_RADIUS |
+-------+--------------------+
```

- Latitude must be between -90 and 90
```sql
mysql> SELECT * FROM poi_s2 WHERE type='cafe' AND S2WITHIN(s2location, 370.547273, 127.047171, 475000);
+-------+----------------------+
| errno | errmsg               |
+-------+----------------------+
|     7 | ERR_INVALID_LATITUDE |
+-------+----------------------+
```

- Longitude must be between -180 and 180
```sql
mysql> SELECT * FROM poi_s2 WHERE type='cafe' AND S2WITHIN(s2location, 37.547273, 1270.047171, 475000);
+-------+-----------------------+
| errno | errmsg                |
+-------+-----------------------+
|     8 | ERR_INVALID_LONGITUDE |
+-------+-----------------------+
```

- S2WITHIN must have 4 arguments at least (maximum 5 arguments)
```sql
mysql> SELECT * FROM poi_s2 WHERE type='cafe' AND S2WITHIN(s2location);
+-------+------------------------+
| errno | errmsg                 |
+-------+------------------------+
|     3 | ERR_INVALID_PARAMETERS |
+-------+------------------------+
```

- S2 geometry does not remove comment of "S2WITHIN(...)" block, so it may disturb parsing arguments of plugin
```sql
-- // Usually mysql client will remove comment of query, in this case no problem.
mysql> SELECT * FROM poi_s2 WHERE type='cafe' AND S2WITHIN(s2location, 37.547273, 127.047171, 475000 /* Location of Ttukseom station */);
+----+----------+------+--------------------+--------------------+---------------------+
| id | name     | type | location_lon       | location_lat       | s2location          |
+----+----------+------+--------------------+--------------------+---------------------+
|  6 | P-Dabang | cafe | 127.04489096350927 | 37.545892027418084 | 3854136371285358265 |
+----+----------+------+--------------------+--------------------+---------------------+

-- // But if you run mysql client with "--comments" option or any other client or connectors, it could make unexpected errors
# mysql -udba -p --comments
mysql> SELECT * FROM poi_s2 WHERE type='cafe' AND S2WITHIN(s2location, 37.547273, 127.047171, 475000 /* Location of Ttukseom station */);
+-------+----------------------+
| errno | errmsg               |
+-------+----------------------+
|    10 | ERR_INVALID_MAXCELLS |
+-------+----------------------+

mysql> SELECT * FROM poi_s2 WHERE type='cafe' AND S2WITHIN(s2location, /* Location of Ttukseom station */ 37.547273, 127.047171, 475000);
+-------+--------------------+
| errno | errmsg             |
+-------+--------------------+
|     9 | ERR_INVALID_RADIUS |
+-------+--------------------+
```


TROUBLE SHOOTING
================
- ERROR 1126 (HY000): Can't open shared library '/db/mysql/lib/plugin/s2_query_rewriter.so' (errno: 13 libgflags.so.2.1: cannot open shared object file: No such file or directory)
   ==> yum install gflags-devel

- ERROR 1126 (HY000): Can't open shared library '/db/mysql/lib/plugin/s2_query_rewriter.so' (errno: 13 libs2.so: cannot open shared object file: No such file or directory)
   ==> copy libs2.so /usr/lib64/

- ERROR 1126 (HY000): Can't open shared library '/db/mysql/lib/plugin/s2_query_rewriter.so' (errno: 13 /lib64/libs2.so: undefined symbol: _ZSt24__throw_out_of_range_fmtPKcz)
   ==> clean all s2/*.o s2/*.so recursively, and rebuild s2 library (libs2.so)

- Data truncation: Out of range value for column 's2cellid' at row 1
     -- // Warning
     -- // s2cell UDF return SIGNED BIGINT (Because MySQL UDF doesn't support return type of UNSIGNED BIGINT)
     -- // Normally, GPS locations located in korea are no problem (Because these are not affected sign bit of BIGINT, e.g. not so big)
     -- // But sometimes, GPS of smartphone return invalid gps location,
     -- // For this reason, you need to casting BIGINT to BIGINT UNSINGED for the safety
     -- //     e.g.) SELECT cast(s2cell(40.6891242,-89.5900406) as unsigned);
     com.mysql.jdbc.MysqlDataTruncation: Data truncation: Out of range value for column 's2cellid' at row 1
     Data truncated for s2cellid : SELECT s2cell(40.6891242,-89.5900406);


LIMITATIONS
============
- Prepared-statement is not supported (Because this plugin will rewrite query before-parse)
- Not allowed using comment in "S2WITHIN" block
- Not tested with big area search which greater than hemisphere
  
TODO
============
- Find a way to inform error to client with MySQL ERROR. (if there's error, current implementation will return different format of result-set)
- Does not parse if there's comment (both of single and multi line) in S2WITHIN() block.


REFERENCEs
============
- https://docs.google.com/presentation/d/1Hl4KapfAENAOf4gv-pSngKwvS_jwNVHRPZTTDzXXn6Q/
- http://blog.christianperone.com/2015/08/googles-s2-geometry-on-the-sphere-cells-and-hilbert-curve/
- https://code.google.com/archive/p/s2-geometry-library/source
