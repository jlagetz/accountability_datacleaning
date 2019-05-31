## Arkansas Voter Registration

This data was obtained Jan. 2019 via public records request.

Number of records: 1,792,963

**Create table from csv file**

See data_dictionary.xls for more information

**Create new columns for clean data**

`ALTER TABLE AR_voters ADD COLUMN RES_CITY_CLEAN; 
ALTER TABLE AR_voters ADD COLUMN YEAR;
ALTER TABLE AR_voters ADD COLUMN BIRTHYEAR;`

**Create lookup table for city to fix inconsistencies**

`CREATE TABLE CITY_LOOKUP AS
SELECT TEXT_RES_CITY,UPPER(TEXT_RES_CITY) AS CITY_CLEAN, COUNT(*)
FROM AR_VOTERS
GROUP BY 1
ORDER BY 1`

**Update main table based on city_lookup**

`UPDATE AR_VOTERS set RES_CITY_CLEAN = (select y.CITY_CLEAN from CITY_LOOKUP as y 
where y.TEXT_RES_CITY=AR_VOTERS.TEXT_RES_CITY)`

**Extract years**

`UPDATE AR_VOTERS set BIRTHYEAR=SUBSTR(date_of_birth,7,4);
UPDATE AR_VOTERS set YEAR=SUBSTR(date_of_registration,7,4)`

**Export table**

`CREATE TABLE AR_VOTERS_OUT AS
Select UPPER(County) AS COUNTY, VoterID, CDE_REGISTRANT_STATUS, CDE_REGISTRANT_REASON, BIRTHYEAR, date_of_registration, 
YEAR, CDE_NAME_TITLE, TEXT_NAME_LAST AS LASTNAME, TEXT_NAME_FIRST AS FIRSTNAME, TEXT_NAME_MIDDLE AS MIDNAME,
CDE_NAME_SUFFIX AS NAME_SUFFIX, TEXT_RES_ADDRESS_NBR AS ADDRESS, TEXT_RES_ADDRESS_NBR_SUFFIX AS ADD_SUFFIX, 
CDE_STREET_DIR_PREFIX AS ADD_DIR_PRE, TEXT_STREET_NAME AS STREET, DESC_STREET_TYPE AS ST_TYPE, 
CDE_STREET_DIR_SUFFIX AS DIR_SUFFIX, TEXT_RES_UNIT_NBR AS UNIT, RES_CITY_CLEAN AS CITY, CDE_RES_STATE AS STATE,TEXT_RES_ZIP5 AS ZIP5, PrecinctName, CDE_PARTY, TEXT_PHONE_AREA_CODE||TEXT_PHONE_EXCHANGE||TEXT_PHONE_LAST_FOUR AS PHONE, TEXT_RES_PHYSICAL_ADDRESS
FROM AR_VOTERS`