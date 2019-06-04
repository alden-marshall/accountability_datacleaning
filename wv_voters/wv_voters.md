## West Virginia voter registration

Obtained via public records request in Nov. 2018 in pipe-delimited file

Number of records: 1,248,580

**Create table**

`CREATE TABLE WV_VOTERS ( ID_VOTER , County_Name , FIRSTNAME , Mid , LASTNAME , Suffix , DATEOFBIRTH , SEX , 
HOUSENO , STREET , STREET2 , UNIT , CITY , STATE , ZIP , MAILHOUSENO , MAILSTREET , MAILSTREET2 , MAILUNIT , 
MAILCITY , MAILSTATE , MAILZIP , REGISTRATIONDATE , PartyAffiliation , Status , CongressionalDistrict , 
SenatorialDistrict , DelegateDistrict , MagisterialDistrict , Precinct_Number , POLL_NAME )`

**Create city lookup table to fix messiness in city names**

`Create table city_lookup as
select city, upper(city), count(*)
from wv_voters
group by 1
order by 1`

**Fix bad ZIPs**

`ALTER TABLE WV_VOTERS ADD COLUMN ZIP5;
UPDATE WV_VOTERS SET ZIP5=ZIP;
UPDATE wv_voters set ZIP5="25882"
WHERE ZIP="00002" AND CITY="MULLENS";
UPDATE wv_voters set 	ZIP5="25661"
WHERE ZIP="00000" AND CITY="WILLIAMSON";
UPDATE wv_voters set 	ZIP5="25958"
WHERE ZIP="00000" AND CITY="ORIENT HILL";
UPDATE wv_voters set 	ZIP5="24986"
WHERE ZIP="00000" AND CITY="NEOLA";
UPDATE wv_voters set 	ZIP5="25918"
WHERE ZIP="00000" AND CITY="SHADY SPRING";
UPDATE wv_voters set 	ZIP5="26680"
WHERE ZIP="00000" AND CITY="RUSSELLVILLE";
UPDATE wv_voters set 	ZIP5="25813"
WHERE ZIP="00000" AND CITY="BEAVER";
`

**Create YEAR column and extract year from REGISTRATIONDATE**

`ALTER TABLE WV_VOTERS ADD COLUMN YEAR;
UPDATE WV_VOTERS SET YEAR=SUBSTR(REGISTRATIONDATE,LENGTH(REGISTRATIONDATE)-11,4);
ALTER TABLE WV_VOTERS ADD COLUMN BIRTHYEAR;
UPDATE WV_VOTERS SET BIRTHYEAR=SUBSTR(DATEOFBIRTH,LENGTH(DATEOFBIRTH)-11,4)` 

**Known issues**

BIRTHDATEs and REGISTRATIONDATEs out of range

**Export**

`REATE TABLE wv_voters_out AS
SELECT ID_VOTER, County_Name, FIRSTNAME, Mid AS MIDDLENAME, LASTNAME, Suffix as SUFFIX, DATEOFBIRTH, 
BIRTHYEAR, SEX, HOUSENO, STREET, STREET2, UNIT, CITY, STATE, ZIP5,   REGISTRATIONDATE, YEAR, 
PartyAffiliation as PARTY, Status, CongressionalDistrict, SenatorialDistrict, DelegateDistrict, 
MagisterialDistrict, Precinct_Number as PRECINCT
FROM wv_voters`