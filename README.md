Here's the translated version of the README.md file, with the formatting intact:

# ADCODE

Chinese administrative division codes, including detailed five-level administrative division codes and geographic fences for county-level and above divisions.

Data comes from the National Bureau of Statistics, Ministry of Civil Affairs, and AutoNavi Map, all of which are public data.

The key point is the geographic location data.

## SYNOPSIS

* Administrative division code data baseline comes from the National Bureau of Statistics: [2016 Statistical Division Codes and Urban-Rural Division Codes (as of July 31, 2016)](http://www.stats.gov.cn/tjsj/tjbz/tjyqhdmhcxhfdm/2016/index.html)
* Administrative division changes are based on announcements from the Ministry of Civil Affairs: [2017 Administrative Division Codes](http://www.mca.gov.cn/article/sj/tjbz/a/2017/)
* Administrative division geographic boundary data comes from: [AutoNavi Map Administrative Divisions](http://lbs.amap.com/api/webservice/guide/api/district)
* Current data time point: July 31, 2016, will be updated with the latest changes in the future.
* Two core data tables:
  * Administrative division data table: `adcode`
  * Geographic fence data table: `fences`
* Data format:
  * [Administrative divisions](data/adcode/) are CSV format data files, split into separate files according to their respective county-level administrative division codes (6 digits).
  * [Geographic fences](data/fences) are JSON format data files, with the geographic fences of the corresponding county-level administrative division codes (6 digits) stored in separate files in GeoJSON format.
  * Text data formats are used for ease of use by other applications and databases, but locally PostgreSQL+PostGIS is actually used. The [`bin/`](bin/) directory provides utility scripts for ETL.

## DESCRIPTION

### Administrative Division Table (adcode)

#### Basic Information

|   Table Name   |                            adcode                            |
| :------------: | :----------------------------------------------------------: |
| Data Directory |                [`data/adcode/`](data/adcode/)                |
| Control Script |                  [`bin/adcode`](bin/adcode)                  |
| Schema Definition |              [`sql/adcode.ddl`](sql/adcode.ddl)              |
| Data Sample |   Chaoyang District：[data/adcode/110105.csv](data/adcode/110105.csv)   |
| Related Knowledge | [Chinese Administrative Division Related Knowledge](https://vonng.com/blog/admin-division/) |

#### Data Structure

| Column       | Type                  | Description                                             |
| ------------ | --------------------- | ------------------------------------------------------- |
| code         | bigint                | 12-digit administrative division code from National Bureau of Statistics |
| parent       | bigint                | 12-digit parent administrative division code            |
| name         | character varying(64) | Administrative unit name                                |
| level        | character varying(16) | Administrative unit level: Country/Province/City/County/Town/Village |
| rank         | integer               | Administrative unit level {0:Country,1:Province,2:City,3:District/County,4:Town/Township,5:Street/Village} |
| adcode       | integer               | 6-digit county-level administrative division code       |
| post_code    | character varying(8)  | Postal code                                             |
| area_code    | character varying(4)  | Long-distance area code                                 |
| ur_code      | character varying(4)  | 3-digit urban-rural classification code                 |
| municipality | boolean               | Whether it's a directly-administered administrative unit |
| virtual      | boolean               | Whether it's a virtual administrative unit, e.g., districts under city jurisdiction |
| dummy        | boolean               | Whether it's a simulated administrative unit, e.g., virtual communities, virtual villages |
| longitude    | double precision      | Geographic center longitude                             |
| latitude     | double precision      | Geographic center latitude                              |
| center       | geometry              | Geographic center, `ST_Point`                           |
| province     | character varying(64) | Province                                                |
| city         | character varying(64) | City                                                    |
| county       | character varying(64) | District/County                                         |
| town         | character varying(64) | Town/Township                                           |
| village      | character varying(64) | Street/Village                                          |

Note that the `center` field uses the PostGIS extension. If you don't want to use PostGIS, you can modify the `center` field type to `TEXT` to maintain compatibility, or use PostgreSQL's built-in `Point` type.

#### Data Sample

```
110105043000	110105000000	East Lake Street Office	town	4	110105	\N	\N	\N	f	f	\N	\N	\N	Beijing	City proper	Chaoyang District	East Lake Street Office	\N
110105400000	110105000000	Capital Airport Street Office	town	4	110105	\N	\N	\N	f	f	\N	\N	\N	Beijing	City proper	Chaoyang District	Capital Airport Street Office	\N
110105035200	110105035000	West Hui Village Committee	village	5	110105	\N	\N	111	f	f	\N	\N	\N	Beijing	City proper	Chaoyang District	Guanzhuang Area Office	West Hui Village Committee
110105035201	110105035000	East Hui Village Committee	village	5	110105	\N	\N	112	f	f	\N	\N	\N	Beijing	City proper	Chaoyang District	Guanzhuang Area Office	East Hui Village Committee
110105001024	110105001000	South Lang Garden Community Residents Committee	village	5	110105	\N	\N	111	f	f	\N	\N	\N	Beijing	City proper	Chaoyang District	Jianwai Street Office	South Lang Garden Community Residents Committee
110105001025	110105001000	North Lang Garden Community Residents Committee	village	5	110105	\N	\N	111	f	f	\N	\N	\N	Beijing	City proper	Chaoyang District	Jianwai Street Office	North Lang Garden Community Residents Committee
```

![](doc/img/adcode-sample.png)

#### Data Description

* Original data comes from public data from the National Bureau of Statistics, Ministry of Civil Affairs, and AutoNavi Map.
* The primary key is `code`, the 12-digit administrative division code used by the National Bureau of Statistics: [Compilation Rules](http://www.stats.gov.cn/tjsj/tjbz/200911/t20091125_8667.html).
* `adcode` is the six-digit county-level administrative division code, and always satisfies `adcode=code/1000000`.
* `level` and `rank` currently have a one-to-one correspondence:

| level |   rank   | code | count  |                        name                        |
| :---: | :------: | :--: | :----: | :------------------------------------------------: |
|   0   | country  |  -   |   1    |                      Country                       |
|   1   | province | 2 digits |   34   | Province, Autonomous Region, Municipality, Special Administrative Region |
|   2   |   city   | 4 digits |  344   | Prefecture-level City, Prefecture, Autonomous Prefecture, League |
|   3   |  county  | 6 digits |  3133  | District, County-level City, County, Autonomous County, Banner, Autonomous Banner, Special District, Forest District |
|   4   |   town   | 9 digits | 42868  | Street, Town, Township, Sumu, Ethnic Township, Ethnic Sumu |
|   5   | village  | 12 digits | 666655 | Residents Committee, Village Committee |

* `post_code` representing postal codes currently has no data and still needs to be supplemented and verified.
* `area_code` representing long-distance area codes still needs to be verified.
* `ur_code` is a three-digit urban-rural classification code. According to the standard, only township and village levels should have this attribute, but in practice, only village-level units have this attribute.
* `municipality` indicates whether the administrative division is directly administered, such as municipalities, directly administered counties, towns, villages, etc.
* `virtual` indicates whether the administrative area is a virtual division. Usually, the superior administrative units of directly administered administrative units are virtual divisions, such as districts under city jurisdiction.
* `dummy` indicates whether the administrative area is a virtual division, but this is another type of virtual, not virtual units created due to direct administration, such as virtual communities.
* `latitude, longitude` are both double-precision floating-point numbers, with 6 significant digits in the original data. All coordinate points use the `GCJ-02` Mars coordinate system.

* The surface type of `center` is PostGIS's `GEOMETRY`, with the underlying type being `ST_Point`.
* `province, city, county, town, village` are denormalized fields added for convenience of use.

* Data for Hong Kong, Macao, and Taiwan at the city and county level needs to be supplemented, and some coordinate system inconsistencies (WGS84 or GCJ-02) need to be corrected.
* This data description is a snapshot as of July 2016. Currently, excluding virtual administrative divisions, 6 outdated counties lack coordinate data, which can be fixed after updating. 83% of towns have coordinate data. Only 8% of village-level administrative units have geographic information. This needs to be improved in the future.

#### Data Quality

Geographic data completeness for non-virtual administrative units:

| level    | total  | have location | miss   |
| -------- | ------ | ------------- | ------ |
| country  | 1      | 1             |        |
| province | 34     | 34            |        |
| city     | 334    | 334           |        |
| county   | 2851   | 2845          | 6      |
| town     | 42846  | 35350         | 7496   |
| village  | 664892 | 59035         | 605857 |

For geographic data at the county level and above, except for 6 county-level divisions (changed in 2016-2017) that lack geographic data, all others are the latest data (as of 2018-02). The six outdated county-level administrative divisions are:

```bash
130421000000	130400000000	Handan County		county	3	130421
139001000000	139000000000	Dingzhou City		county	3	139001
139002000000	139000000000	Xinji City		county	3	139002
320812000000	320800000000	Qingjiangpu District	county	3	320812
330204000000	330200000000	Jiangdong District		county	3	330204
410211000000	410200000000	Jinming District		county	3	410211
```

#### Utility Scripts

Data tables have the convenience of operational management, while text files have the convenience of readable communication. Therefore, practical ETL tools are provided: [`bin/adcode`](bin/adcode)

Most functions require a PostgreSQL instance with superuser privileges and a database with the PostGIS extension installed (not mandatory). Connection parameters can be provided by specifying the `PGURL` environment variable. The default is `postgres://localhost:5432/adcode`

```bash
Usage:
	bin/adcode <action>
	
	action list:

	create	:	Create adcode table, if it already exists, the original table will be deleted.
	index	:	Create indexes on the adcode table (note that a large number of indexes will severely slow down insertion speed during large-scale insertions).
	order	:	Sort and rebuild adcode to make it physically ordered (rank,code ASC)
	drop	:	Delete the fences table
	trunc	:	Empty the adcode table
	clean	:	Clear the dumps in data/adcode
	dump	:	Dump the adcode table as text to data/adcode/, if adcode list parameters are provided, only dump the corresponding data
	load	:	Import from text dumps in data/adcode/ to the adcode table, if adcode list parameters are provided, only import the corresponding data
	backup	:	Backup adcode to data/backup/adcode.sql
	restore	:	Restore adcode from backup data/backup/adcode.sql
	check	:	Check adcode data
	reload	:	Empty the adcode table and reload data from data/adcode
	reset	:	Delete and create the adcode table
	setup	:	Initialize: create table, load data, create indexes.
	usage	:	Display usage of the adcode control script
```

When used locally, you can use `backup/restore` for backup and recovery, which will be much faster (a few seconds).

When submitting PR/PULL and releases, use `dump/load` to generate text format data (20-30 seconds).

This table will hardly have any writes in normal times, so a large number of indexes have been created, which can greatly speed up queries and dumps.

However, during large-scale loading, indexes may severely slow down insertion speed, you may need to consider rebuilding indexes after import.

### Geographic Fence Table (fences)

Note: According to the Surveying and Mapping Law of the People's Republic of China.

#### Basic Information

| Table Name | fences                                                       |
| ---------- | ------------------------------------------------------------ |
| Data Directory | [`data/fences/`](data/fences/)                               |
| Control Script | [`bin/fences`](bin/fences)                                   |
| Schema Definition | [`sql/fences.ddl`](sql/fences.ddl)                           |
| Data Sample | Chaoyang District Geographic Fence：[data/fences/110105.json](data/fences/110105.json) |
| Related Knowledge | [GCJ-02](https://en.wikipedia.org/wiki/Restrictions_on_geographic_data_in_China#GCJ-02), [PostGIS Geometry](http://workshops.boundlessgeo.com/postgis-intro/geometries.html), [GeoJSON](http://geojson.org) |

#### Data Structure

| Column |   Type   |         Description          |
|--------|----------|------------------------------|
| code   | bigint   | 12-digit administrative division code from National Bureau of Statistics |
| adcode | integer  | 6-digit county-level administrative division code |
| fence  | geometry | Geographic fence, GCJ-02, MultiPolygon |

#### Data Sample

For example, the [geographic fence](data/fences/820008.json) for the Parish of St. Francis Xavier in Macau Special Administrative Region, with a 6-digit county-level administrative division code of 820008, is:

```json
{"type":"Polygon","coordinates":[[[113.601946,22.138113],[113.603872,22.132865],[113.603681,22.132371],[113.597564,22.125115],[113.580194,22.109694],[113.578955,22.10893],[113.576328,22.108147],[113.561046,22.106214],[113.559643,22.106216],[113.553703,22.107469],[113.55353,22.110847],[113.553296,22.116852],[113.553297,22.120642],[113.553574,22.125144],[113.563949,22.127289],[113.563972,22.127298],[113.564018,22.127343],[113.56407,22.127443],[113.564391,22.128297],[113.564475,22.128366],[113.56446,22.128395],[113.564529,22.128445],[113.564552,22.128452],[113.564651,22.128477],[113.564798,22.128486],[113.567266,22.128386],[113.567475,22.128416],[113.56763,22.128485],[113.567778,22.128615],[113.567856,22.128713],[113.56789,22.128884],[113.568003,22.131992],[113.568107,22.132224],[113.568324,22.13235],[113.568618,22.132386],[113.571408,22.132306],[113.572257,22.132331],[113.573184,22.132482],[113.574068,22.132806],[113.574848,22.133293],[113.575827,22.134051],[113.57659,22.134553],[113.577257,22.134854],[113.57802,22.135012],[113.579329,22.135134],[113.580612,22.135239],[113.581375,22.135379],[113.582311,22.135821],[113.583056,22.136473],[113.584062,22.137923],[113.589178,22.14428],[113.601946,22.138113]]]}
```

![img](doc/img/fences-sample.png)

#### Data Description

* The `fences` table can be joined with the `adcode` table via `code`.
* `fence` uses PostGIS's `GEOMETRY` type for storage, with the underlying type being `POLYGON` or `MULTIPOLYGON` (with exclaves)
* About 300 counties lack fence data, to be corrected in the future.
* The geographic fences of some virtual administrative divisions should theoretically be the union of their sub-administrative divisions, which also needs to be corrected.

#### Utility Scripts

The usage of `bin/fences` is the same as `bin/adcode`

```bash
Usage:
	bin/fences <action>
	
	action list:

	create	:	Create fences table, if it already exists, the original table will be deleted.
	index	:	Create indexes on the fences table (note that a large number of indexes will severely slow down insertion speed during large-scale insertions).
	order	:	Sort and rebuild fences to make it physically ordered (rank,code ASC)
	drop	:	Delete the fences table
	trunc	:	Empty the fences table
	clean	:	Clear the dumps in data/fences
	dump	:	Dump the fences table as text to data/fences/, if adcode list parameters are provided, only dump the corresponding data
	load	:	Import from text dumps in data/fences/ to the fences table, if adcode list parameters are provided, only import the corresponding data
	backup	:	Backup fences to data/backup/fences.sql
	restore	:	Restore fences from backup data/backup/fences.sql
	check	:	Check fences data
	reload	:	Empty the fences table and reload data from data/fences
	reset	:	Delete and create the fences table
	setup	:	Initialize: create table, load data, create indexes.
	usage	:	Display usage of the fences control script
```

### Others

A [makefile](makefile) is provided in the project's root directory, which can perform the same operations on both tables simultaneously.

```bash
# Create database
make createdb

# Create, load, build indexes
make setup
```

## ISSUES

* Using the latest data from the National Bureau of Statistics as of July 31, 2016, some division information is outdated and needs to be supplemented and corrected in the future.
* For geographic fences, some newly added counties at the county level and above will lack data. To be supplemented in the future.
* Data for Hong Kong, Macao, and Taiwan regions needs to be supplemented, as there are no publicly available official administrative division codes below the provincial level.
* Postal codes and long-distance area code data need to be verified.

## CONTRIBUTION

Contributions to this project are welcome. Changes to division data should include: links to announcements from the Ministry of Civil Affairs, preferably with changes given in SQL form.

PRs can make changes directly in `adcode/*.csv` and `fences/*.json`, and the changes should be clearly visible from the commit log, without adjusting the format.

Please note that any PR merged into master needs to ensure data consistency, including:

* The prefixes of sub-administrative divisions always remain consistent with their parent administrative divisions, including `code, parent`.
* The denormalized fields `adcode, province, city, county, town` of sub-administrative divisions in the `adcode` table need to be modified synchronously.
* Synchronous changes need to be made in the `fences` table, including changes to `code, adcode`. Note that changes to fences may lead to changes in the fences of virtual parent-level administrative divisions.
* Any changes need to be reflected in the commit log, including the changes made, the time of change, referenced announcements, etc.

Currently, I will catch up with the progress of the Ministry of Civil Affairs when I have time, but this work is really boring and uninteresting.

## LICENSE

![](https://i.creativecommons.org/l/by/4.0/88x31.png)

This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.