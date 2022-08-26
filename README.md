# American Geography
Where are the fastest growing counties in the country? Where are the youngest cities? The oldest? Where are the most expensive neighborhoods? What can street data tell us about these places? This is my deep dive into US census and street data.

1. [Importing](#1-importing)
2. [Processing](#2-processing)
2. [Exporting](#3-exporting)

## 1. Importing

Census geography files
```bash
# national level
ogr2ogr -overwrite -skipfailures --config PG_USE_COPY YES -f PGDump -t_srs "EPSG:3857" /vsistdout/ tlgdb_2021_a_us_nationgeo.gdb | psql -d us -f -

# substate level
ogr2ogr -overwrite -skipfailures --config PG_USE_COPY YES -f PGDump -t_srs "EPSG:3857" /vsistdout/ tlgdb_2021_a_us_substategeo.gdb | psql -d us -f -

# union census designated places and incorporated places
psql -d us -c "CREATE TABLE place AS (SELECT * FROM incorporated_place UNION ALL SELECT * FROM census_designated_place);"

# merge state puma files then import
ogrmerge.py -overwrite_ds -single -nln puma -o puma.gpkg $(ls *.zip | sed 's/^/\/vsizip\//g' | paste -sd' ')
ogr2ogr -overwrite -skipfailures -nlt promote_to_multi --config PG_USE_COPY YES -f PGDump -t_srs "EPSG:3857" /vsistdout/ puma.gpkg | psql -d us -f -
```

Census population tables
```bash
iconv -f latin1 -t ascii//TRANSLIT NST-EST2021-alldata.csv > NST-EST2021-alldata_iconv.csv
psql -d us -c "CREATE TABLE nst_est2021($(head -1 NST-EST2021-alldata_iconv.csv | sed -e 's/,/ VARCHAR,/g' -e 's/\r$/ VARCHAR/g'));"
psql -d us -c "\COPY nst_est2021 FROM 'NST-EST2021-alldata_iconv.csv' WITH CSV HEADER;"
```

Census data tables
```bash
# todo: 2019-2010
table='dp05_state_2020'
file='ACSDP5Y2020.DP05_data_with_overlays_2022-07-26T084339.csv'
iconv -f latin1 -t ascii//TRANSLIT ${file} | awk 'NR!=2' > ${file%.*}_iconv.csv
psql -d us -c "CREATE TABLE ${table}($(head -1 ${file} | sed -e 's/"//g' -e 's/,/ VARCHAR,/g' -e 's/\r$/ VARCHAR/g'));"
psql -d us -c "\COPY ${table} FROM ${file%.*}_iconv.csv WITH CSV HEADER;"
```

GeoNames
```bash
ogr2ogr -overwrite -skipfailures --config PG_USE_COPY YES -f PGDump -t_srs 'EPSG:3857' -nln geonames_us -where "countrycode = 'US'" /vsistdout/ PG:dbname=world geonames | psql -d us -f -
```

OpenStreetMap
```bash
ogr2ogr -overwrite -skipfailures --config OSM_MAX_TMPFILE_SIZE 1000 --config OGR_INTERLEAVED_READING YES --config PG_USE_COPY YES -f PGDump -t_srs "EPSG:3857" -nln points_us /vsistdout/ us-latest.osm.pbf points | psql -d us -f -
```

NaturalEarth
```bash
ogr2ogr -overwrite -skipfailures --config PG_USE_COPY YES -f PGDump -t_srs "EPSG:3857" -nlt promote_to_multi /vsistdout/ natural_earth_vector_3857.gpkg ne_10m_admin_2_counties | psql -d us -f -
```

## 2. Processing

Choose interesting sounding columns from ACS datatables, mostly percentages.
```bash
# us
psql -d us -c "CREATE TABLE us2020 AS SELECT a.*, DP05_0001E AS pop2020, DP05_0005PE AS age_5_under, DP05_0006PE AS age_5_9, DP05_0007PE AS age_10_14, DP05_0008PE AS age_15_19, DP05_0009PE AS age_20_24, DP05_0010PE AS age_25_34, DP05_0011PE AS age_35_44, DP05_0012PE AS age_45_54, DP05_0013PE AS age_55_59, DP05_0014PE AS age_60_64, DP05_0015PE AS age_65_74, DP05_0016PE AS age_75_84, DP05_0017PE AS age_85_over, DP05_0018E AS age_median, DP05_0019PE AS age_under18, DP05_0020PE AS age_16_over, DP05_0021PE AS age_18_over, DP05_0022PE AS age_21_over, DP05_0023PE AS age_62_over, DP05_0024PE AS age_65_over, DP05_0035PE AS race_mixed, DP05_0037PE AS race_white, DP05_0038PE AS race_black, DP05_0039PE AS race_native, DP05_0044PE AS race_asian, DP05_0052PE AS race_pacific, DP05_0071PE AS race_hispanic, DP05_0086E AS housing_total, DP03_0004PE AS employed, DP03_0005PE AS unemployed, DP03_0027PE AS job_bus_sci_art, DP03_0028PE AS job_service, DP03_0029PE AS job_sales, DP03_0030PE AS job_construction, DP03_0031PE AS job_prod_transpo, DP03_0033PE AS in_agri_mining, DP03_0034PE AS in_construction, DP03_0035PE AS in_manufacturing, DP03_0036PE AS in_wholesale, DP03_0037PE AS in_retail, DP03_0038PE AS in_transportation, DP03_0039PE AS in_information, DP03_0040PE AS in_finance, DP03_0041PE AS in_professional, DP03_0042PE AS in_education_health, DP03_0043PE AS in_arts_food, DP03_0044PE AS in_other, DP03_0045PE AS in_public_admin, DP03_0047PE AS class_private, DP03_0048PE AS class_govt, DP03_0049PE AS class_selfemployed, DP03_0050PE AS class_unpaid, DP03_0052PE AS income_10000_less, DP03_0053PE AS income_10000_14999, DP03_0054PE AS income_15000_24999, DP03_0055PE AS income_25000_34999, DP03_0056PE AS income_35000_49999, DP03_0057PE AS income_50000_74999, DP03_0058PE AS income_75000_99999, DP03_0059PE AS income_100000_149999, DP03_0060PE AS income_150000_199999, DP03_0061PE AS income_200000_more, DP03_0062E AS income_median, DP03_0063E AS income_mean, DP03_0088E AS income_percapita, DP03_0092E AS earnings_median, DP03_0093E AS earnings_male, DP03_0094E AS earnings_female, DP04_0001E AS housing_units, DP04_0002PE AS housing_occupied, DP04_0003PE AS housing_vacant, DP04_0004PE AS vacancy_homeowner, DP04_0005PE AS vacancy_rental, DP04_0007PE AS units_1_detached, DP04_0008PE AS units_1_attached, DP04_0009PE AS units_2, DP04_0010PE AS units_3_4, DP04_0011PE AS units_5_9, DP04_0012PE AS units_10_19, DP04_0013PE AS units_20_more, DP04_0014PE AS units_mobile, DP04_0015PE AS units_boat_rv_van, DP04_0017PE AS built_2014_later, DP04_0018PE AS built_2010_2013, DP04_0019PE AS built_2000_2009, DP04_0020PE AS built_1990_1999, DP04_0021PE AS built_1980_1989, DP04_0022PE AS built_1970_1979, DP04_0023PE AS built_1960_1969, DP04_0024PE AS built_1950_1959, DP04_0025PE AS built_1940_1949, DP04_0026PE AS built_1939_earlier, DP04_0028PE AS rooms_1, DP04_0029PE AS rooms_2, DP04_0030PE AS rooms_3, DP04_0031PE AS rooms_4, DP04_0032PE AS rooms_5, DP04_0033PE AS rooms_6, DP04_0034PE AS rooms_7, DP04_0035PE AS rooms_8, DP04_0036PE AS rooms_9, DP04_0037E AS rooms_median, DP04_0046PE AS occupied_owner, DP04_0047PE AS occupied_renter, DP04_0081PE AS value_50000_less, DP04_0082PE AS value_50000_99999, DP04_0083PE AS value_100000_149000, DP04_0084PE AS value_150000_199999, DP04_0085PE AS value_200000_299999, DP04_0086PE AS value_300000_499999, DP04_0087PE AS value_500000_999999, DP04_0088PE AS value_1000000_more, DP04_0089E AS value_median, DP04_0127PE AS rent_500_less, DP04_0128PE AS rent_500_999, DP04_0129PE AS rent_1000_1499, DP04_0130PE AS rent_1500_1999, DP04_0131PE AS rent_2000_2499, DP04_0132PE AS rent_2500_2999, DP04_0133PE AS rent_3000_more, DP04_0134E AS rent_median, DP04_0137PE AS grapi_15_less, DP04_0138PE AS grapi_15_19, DP04_0139PE AS grapi_20_24, DP04_0140PE AS grapi_25_29, DP04_0141PE AS grapi_30_34, DP04_0142PE AS grapi_35_more, DP02_0060PE AS edu_9_less, DP02_0061PE AS edu_9_12, DP02_0062PE AS edu_highschool, DP02_0063PE AS edu_college_nodegree, DP02_0064PE AS edu_associate_degree, DP02_0065PE AS edu_bachelor_degree, DP02_0066PE AS edu_grad_degree, DP02_0067PE AS edu_highschool_higher, DP02_0068PE AS edu_bachelor_higher, DP02_0125PE AS ancestry_american, DP02_0126PE AS ancestry_arab, DP02_0127PE AS ancestry_czech, DP02_0128PE AS ancestry_danish, DP02_0129PE AS ancestry_dutch, DP02_0130PE AS ancestry_english, DP02_0131PE AS ancestry_french, DP02_0132PE AS ancestry_frenchcanadian, DP02_0133PE AS ancestry_german, DP02_0134PE AS ancestry_greek, DP02_0135PE AS ancestry_hungarian, DP02_0136PE AS ancestry_irish, DP02_0137PE AS ancestry_italian, DP02_0138PE AS ancestry_lithuanian, DP02_0139PE AS ancestry_norwegian, DP02_0140PE AS ancestry_polish, DP02_0141PE AS ancestry_portuguese, DP02_0142PE AS ancestry_russian, DP02_0143PE AS ancestry_scottish_irish, DP02_0144PE AS ancestry_scottish, DP02_0145PE AS ancestry_slovak, DP02_0146PE AS ancestry_subsaharan_african, DP02_0147PE AS ancestry_swedish, DP02_0148PE AS ancestry_swiss, DP02_0149PE AS ancestry_ukranian, DP02_0150PE AS ancestry_welsh, DP02_0151PE AS ancestry_west_indian FROM nst_est2021 a, dp02_us_2020 b, dp03_us_2020 c, dp04_us_2020 d, dp05_us_2020 e WHERE a.name = 'United States';"

# states
psql -d us -c "CREATE TABLE state2020 AS SELECT a.\"SHAPE\", a.geoid, a.name, DP05_0001E AS pop2020, DP05_0005PE AS age_5_under, DP05_0006PE AS age_5_9, DP05_0007PE AS age_10_14, DP05_0008PE AS age_15_19, DP05_0009PE AS age_20_24, DP05_0010PE AS age_25_34, DP05_0011PE AS age_35_44, DP05_0012PE AS age_45_54, DP05_0013PE AS age_55_59, DP05_0014PE AS age_60_64, DP05_0015PE AS age_65_74, DP05_0016PE AS age_75_84, DP05_0017PE AS age_85_over, DP05_0018E AS age_median, DP05_0019PE AS age_under18, DP05_0020PE AS age_16_over, DP05_0021PE AS age_18_over, DP05_0022PE AS age_21_over, DP05_0023PE AS age_62_over, DP05_0024PE AS age_65_over, DP05_0035PE AS race_mixed, DP05_0037PE AS race_white, DP05_0038PE AS race_black, DP05_0039PE AS race_native, DP05_0044PE AS race_asian, DP05_0052PE AS race_pacific, DP05_0071PE AS race_hispanic, DP05_0086E AS housing_total, DP03_0004PE AS employed, DP03_0005PE AS unemployed, DP03_0027PE AS job_bus_sci_art, DP03_0028PE AS job_service, DP03_0029PE AS job_sales, DP03_0030PE AS job_construction, DP03_0031PE AS job_prod_transpo, DP03_0033PE AS in_agri_mining, DP03_0034PE AS in_construction, DP03_0035PE AS in_manufacturing, DP03_0036PE AS in_wholesale, DP03_0037PE AS in_retail, DP03_0038PE AS in_transportation, DP03_0039PE AS in_information, DP03_0040PE AS in_finance, DP03_0041PE AS in_professional, DP03_0042PE AS in_education_health, DP03_0043PE AS in_arts_food, DP03_0044PE AS in_other, DP03_0045PE AS in_public_admin, DP03_0047PE AS class_private, DP03_0048PE AS class_govt, DP03_0049PE AS class_selfemployed, DP03_0050PE AS class_unpaid, DP03_0052PE AS income_10000_less, DP03_0053PE AS income_10000_14999, DP03_0054PE AS income_15000_24999, DP03_0055PE AS income_25000_34999, DP03_0056PE AS income_35000_49999, DP03_0057PE AS income_50000_74999, DP03_0058PE AS income_75000_99999, DP03_0059PE AS income_100000_149999, DP03_0060PE AS income_150000_199999, DP03_0061PE AS income_200000_more, DP03_0062E AS income_median, DP03_0063E AS income_mean, DP03_0088E AS income_percapita, DP03_0092E AS earnings_median, DP03_0093E AS earnings_male, DP03_0094E AS earnings_female, DP04_0001E AS housing_units, DP04_0002PE AS housing_occupied, DP04_0003PE AS housing_vacant, DP04_0004PE AS vacancy_homeowner, DP04_0005PE AS vacancy_rental, DP04_0007PE AS units_1_detached, DP04_0008PE AS units_1_attached, DP04_0009PE AS units_2, DP04_0010PE AS units_3_4, DP04_0011PE AS units_5_9, DP04_0012PE AS units_10_19, DP04_0013PE AS units_20_more, DP04_0014PE AS units_mobile, DP04_0015PE AS units_boat_rv_van, DP04_0017PE AS built_2014_later, DP04_0018PE AS built_2010_2013, DP04_0019PE AS built_2000_2009, DP04_0020PE AS built_1990_1999, DP04_0021PE AS built_1980_1989, DP04_0022PE AS built_1970_1979, DP04_0023PE AS built_1960_1969, DP04_0024PE AS built_1950_1959, DP04_0025PE AS built_1940_1949, DP04_0026PE AS built_1939_earlier, DP04_0028PE AS rooms_1, DP04_0029PE AS rooms_2, DP04_0030PE AS rooms_3, DP04_0031PE AS rooms_4, DP04_0032PE AS rooms_5, DP04_0033PE AS rooms_6, DP04_0034PE AS rooms_7, DP04_0035PE AS rooms_8, DP04_0036PE AS rooms_9, DP04_0037E AS rooms_median, DP04_0046PE AS occupied_owner, DP04_0047PE AS occupied_renter, DP04_0081PE AS value_50000_less, DP04_0082PE AS value_50000_99999, DP04_0083PE AS value_100000_149000, DP04_0084PE AS value_150000_199999, DP04_0085PE AS value_200000_299999, DP04_0086PE AS value_300000_499999, DP04_0087PE AS value_500000_999999, DP04_0088PE AS value_1000000_more, DP04_0089E AS value_median, DP04_0127PE AS rent_500_less, DP04_0128PE AS rent_500_999, DP04_0129PE AS rent_1000_1499, DP04_0130PE AS rent_1500_1999, DP04_0131PE AS rent_2000_2499, DP04_0132PE AS rent_2500_2999, DP04_0133PE AS rent_3000_more, DP04_0134E AS rent_median, DP04_0137PE AS grapi_15_less, DP04_0138PE AS grapi_15_19, DP04_0139PE AS grapi_20_24, DP04_0140PE AS grapi_25_29, DP04_0141PE AS grapi_30_34, DP04_0142PE AS grapi_35_more, DP02_0060PE AS edu_9_less, DP02_0061PE AS edu_9_12, DP02_0062PE AS edu_highschool, DP02_0063PE AS edu_college_nodegree, DP02_0064PE AS edu_associate_degree, DP02_0065PE AS edu_bachelor_degree, DP02_0066PE AS edu_grad_degree, DP02_0067PE AS edu_highschool_higher, DP02_0068PE AS edu_bachelor_higher, DP02_0125PE AS ancestry_american, DP02_0126PE AS ancestry_arab, DP02_0127PE AS ancestry_czech, DP02_0128PE AS ancestry_danish, DP02_0129PE AS ancestry_dutch, DP02_0130PE AS ancestry_english, DP02_0131PE AS ancestry_french, DP02_0132PE AS ancestry_frenchcanadian, DP02_0133PE AS ancestry_german, DP02_0134PE AS ancestry_greek, DP02_0135PE AS ancestry_hungarian, DP02_0136PE AS ancestry_irish, DP02_0137PE AS ancestry_italian, DP02_0138PE AS ancestry_lithuanian, DP02_0139PE AS ancestry_norwegian, DP02_0140PE AS ancestry_polish, DP02_0141PE AS ancestry_portuguese, DP02_0142PE AS ancestry_russian, DP02_0143PE AS ancestry_scottish_irish, DP02_0144PE AS ancestry_scottish, DP02_0145PE AS ancestry_slovak, DP02_0146PE AS ancestry_subsaharan_african, DP02_0147PE AS ancestry_swedish, DP02_0148PE AS ancestry_swiss, DP02_0149PE AS ancestry_ukranian, DP02_0150PE AS ancestry_welsh, DP02_0151PE AS ancestry_west_indian FROM state a, dp02_state_2020 b, dp03_state_2020 c, dp04_state_2020 d, dp05_state_2020 e WHERE a.geoid = SUBSTRING(b.geo_id,10,2) AND a.geoid = SUBSTRING(c.geo_id,10,2) AND a.geoid = SUBSTRING(d.geo_id,10,2) AND a.geoid = SUBSTRING(e.geo_id,10,2) AND a.sumlev = '040';"

# counties
psql -d us -c "CREATE TABLE county2020 AS SELECT a.\"SHAPE\", a.geoid, a.namelsad AS name, DP05_0001E AS pop2020, DP05_0005PE AS age_5_under, DP05_0006PE AS age_5_9, DP05_0007PE AS age_10_14, DP05_0008PE AS age_15_19, DP05_0009PE AS age_20_24, DP05_0010PE AS age_25_34, DP05_0011PE AS age_35_44, DP05_0012PE AS age_45_54, DP05_0013PE AS age_55_59, DP05_0014PE AS age_60_64, DP05_0015PE AS age_65_74, DP05_0016PE AS age_75_84, DP05_0017PE AS age_85_over, DP05_0018E AS age_median, DP05_0019PE AS age_under18, DP05_0020PE AS age_16_over, DP05_0021PE AS age_18_over, DP05_0022PE AS age_21_over, DP05_0023PE AS age_62_over, DP05_0024PE AS age_65_over, DP05_0035PE AS race_mixed, DP05_0037PE AS race_white, DP05_0038PE AS race_black, DP05_0039PE AS race_native, DP05_0044PE AS race_asian, DP05_0052PE AS race_pacific, DP05_0071PE AS race_hispanic, DP05_0086E AS housing_total, DP03_0004PE AS employed, DP03_0005PE AS unemployed, DP03_0027PE AS job_bus_sci_art, DP03_0028PE AS job_service, DP03_0029PE AS job_sales, DP03_0030PE AS job_construction, DP03_0031PE AS job_prod_transpo, DP03_0033PE AS in_agri_mining, DP03_0034PE AS in_construction, DP03_0035PE AS in_manufacturing, DP03_0036PE AS in_wholesale, DP03_0037PE AS in_retail, DP03_0038PE AS in_transportation, DP03_0039PE AS in_information, DP03_0040PE AS in_finance, DP03_0041PE AS in_professional, DP03_0042PE AS in_education_health, DP03_0043PE AS in_arts_food, DP03_0044PE AS in_other, DP03_0045PE AS in_public_admin, DP03_0047PE AS class_private, DP03_0048PE AS class_govt, DP03_0049PE AS class_selfemployed, DP03_0050PE AS class_unpaid, DP03_0052PE AS income_10000_less, DP03_0053PE AS income_10000_14999, DP03_0054PE AS income_15000_24999, DP03_0055PE AS income_25000_34999, DP03_0056PE AS income_35000_49999, DP03_0057PE AS income_50000_74999, DP03_0058PE AS income_75000_99999, DP03_0059PE AS income_100000_149999, DP03_0060PE AS income_150000_199999, DP03_0061PE AS income_200000_more, DP03_0062E AS income_median, DP03_0063E AS income_mean, DP03_0088E AS income_percapita, DP03_0092E AS earnings_median, DP03_0093E AS earnings_male, DP03_0094E AS earnings_female, DP04_0001E AS housing_units, DP04_0002PE AS housing_occupied, DP04_0003PE AS housing_vacant, DP04_0004PE AS vacancy_homeowner, DP04_0005PE AS vacancy_rental, DP04_0007PE AS units_1_detached, DP04_0008PE AS units_1_attached, DP04_0009PE AS units_2, DP04_0010PE AS units_3_4, DP04_0011PE AS units_5_9, DP04_0012PE AS units_10_19, DP04_0013PE AS units_20_more, DP04_0014PE AS units_mobile, DP04_0015PE AS units_boat_rv_van, DP04_0017PE AS built_2014_later, DP04_0018PE AS built_2010_2013, DP04_0019PE AS built_2000_2009, DP04_0020PE AS built_1990_1999, DP04_0021PE AS built_1980_1989, DP04_0022PE AS built_1970_1979, DP04_0023PE AS built_1960_1969, DP04_0024PE AS built_1950_1959, DP04_0025PE AS built_1940_1949, DP04_0026PE AS built_1939_earlier, DP04_0028PE AS rooms_1, DP04_0029PE AS rooms_2, DP04_0030PE AS rooms_3, DP04_0031PE AS rooms_4, DP04_0032PE AS rooms_5, DP04_0033PE AS rooms_6, DP04_0034PE AS rooms_7, DP04_0035PE AS rooms_8, DP04_0036PE AS rooms_9, DP04_0037E AS rooms_median, DP04_0046PE AS occupied_owner, DP04_0047PE AS occupied_renter, DP04_0081PE AS value_50000_less, DP04_0082PE AS value_50000_99999, DP04_0083PE AS value_100000_149000, DP04_0084PE AS value_150000_199999, DP04_0085PE AS value_200000_299999, DP04_0086PE AS value_300000_499999, DP04_0087PE AS value_500000_999999, DP04_0088PE AS value_1000000_more, DP04_0089E AS value_median, DP04_0127PE AS rent_500_less, DP04_0128PE AS rent_500_999, DP04_0129PE AS rent_1000_1499, DP04_0130PE AS rent_1500_1999, DP04_0131PE AS rent_2000_2499, DP04_0132PE AS rent_2500_2999, DP04_0133PE AS rent_3000_more, DP04_0134E AS rent_median, DP04_0137PE AS grapi_15_less, DP04_0138PE AS grapi_15_19, DP04_0139PE AS grapi_20_24, DP04_0140PE AS grapi_25_29, DP04_0141PE AS grapi_30_34, DP04_0142PE AS grapi_35_more, DP02_0060PE AS edu_9_less, DP02_0061PE AS edu_9_12, DP02_0062PE AS edu_highschool, DP02_0063PE AS edu_college_nodegree, DP02_0064PE AS edu_associate_degree, DP02_0065PE AS edu_bachelor_degree, DP02_0066PE AS edu_grad_degree, DP02_0067PE AS edu_highschool_higher, DP02_0068PE AS edu_bachelor_higher, DP02_0125PE AS ancestry_american, DP02_0126PE AS ancestry_arab, DP02_0127PE AS ancestry_czech, DP02_0128PE AS ancestry_danish, DP02_0129PE AS ancestry_dutch, DP02_0130PE AS ancestry_english, DP02_0131PE AS ancestry_french, DP02_0132PE AS ancestry_frenchcanadian, DP02_0133PE AS ancestry_german, DP02_0134PE AS ancestry_greek, DP02_0135PE AS ancestry_hungarian, DP02_0136PE AS ancestry_irish, DP02_0137PE AS ancestry_italian, DP02_0138PE AS ancestry_lithuanian, DP02_0139PE AS ancestry_norwegian, DP02_0140PE AS ancestry_polish, DP02_0141PE AS ancestry_portuguese, DP02_0142PE AS ancestry_russian, DP02_0143PE AS ancestry_scottish_irish, DP02_0144PE AS ancestry_scottish, DP02_0145PE AS ancestry_slovak, DP02_0146PE AS ancestry_subsaharan_african, DP02_0147PE AS ancestry_swedish, DP02_0148PE AS ancestry_swiss, DP02_0149PE AS ancestry_ukranian, DP02_0150PE AS ancestry_welsh, DP02_0151PE AS ancestry_west_indian FROM county a, dp02_county_2020 b, dp03_county_2020 c, dp04_county_2020 d, dp05_county_2020 e WHERE a.geoid = SUBSTRING(b.geo_id,10,5) AND a.geoid = SUBSTRING(c.geo_id,10,5) AND a.geoid = SUBSTRING(d.geo_id,10,5) AND a.geoid = SUBSTRING(e.geo_id,10,5);"

# places
psql -d us -c "CREATE TABLE place2020 AS SELECT a.\"SHAPE\", a.geoid, a.namelsad as name, DP05_0001E AS pop2020, DP05_0005PE AS age_5_under, DP05_0006PE AS age_5_9, DP05_0007PE AS age_10_14, DP05_0008PE AS age_15_19, DP05_0009PE AS age_20_24, DP05_0010PE AS age_25_34, DP05_0011PE AS age_35_44, DP05_0012PE AS age_45_54, DP05_0013PE AS age_55_59, DP05_0014PE AS age_60_64, DP05_0015PE AS age_65_74, DP05_0016PE AS age_75_84, DP05_0017PE AS age_85_over, DP05_0018E AS age_median, DP05_0019PE AS age_under18, DP05_0020PE AS age_16_over, DP05_0021PE AS age_18_over, DP05_0022PE AS age_21_over, DP05_0023PE AS age_62_over, DP05_0024PE AS age_65_over, DP05_0035PE AS race_mixed, DP05_0037PE AS race_white, DP05_0038PE AS race_black, DP05_0039PE AS race_native, DP05_0044PE AS race_asian, DP05_0052PE AS race_pacific, DP05_0071PE AS race_hispanic, DP05_0086E AS housing_total, DP03_0004PE AS employed, DP03_0005PE AS unemployed, DP03_0027PE AS job_bus_sci_art, DP03_0028PE AS job_service, DP03_0029PE AS job_sales, DP03_0030PE AS job_construction, DP03_0031PE AS job_prod_transpo, DP03_0033PE AS in_agri_mining, DP03_0034PE AS in_construction, DP03_0035PE AS in_manufacturing, DP03_0036PE AS in_wholesale, DP03_0037PE AS in_retail, DP03_0038PE AS in_transportation, DP03_0039PE AS in_information, DP03_0040PE AS in_finance, DP03_0041PE AS in_professional, DP03_0042PE AS in_education_health, DP03_0043PE AS in_arts_food, DP03_0044PE AS in_other, DP03_0045PE AS in_public_admin, DP03_0047PE AS class_private, DP03_0048PE AS class_govt, DP03_0049PE AS class_selfemployed, DP03_0050PE AS class_unpaid, DP03_0052PE AS income_10000_less, DP03_0053PE AS income_10000_14999, DP03_0054PE AS income_15000_24999, DP03_0055PE AS income_25000_34999, DP03_0056PE AS income_35000_49999, DP03_0057PE AS income_50000_74999, DP03_0058PE AS income_75000_99999, DP03_0059PE AS income_100000_149999, DP03_0060PE AS income_150000_199999, DP03_0061PE AS income_200000_more, DP03_0062E AS income_median, DP03_0063E AS income_mean, DP03_0088E AS income_percapita, DP03_0092E AS earnings_median, DP03_0093E AS earnings_male, DP03_0094E AS earnings_female, DP04_0001E AS housing_units, DP04_0002PE AS housing_occupied, DP04_0003PE AS housing_vacant, DP04_0004PE AS vacancy_homeowner, DP04_0005PE AS vacancy_rental, DP04_0007PE AS units_1_detached, DP04_0008PE AS units_1_attached, DP04_0009PE AS units_2, DP04_0010PE AS units_3_4, DP04_0011PE AS units_5_9, DP04_0012PE AS units_10_19, DP04_0013PE AS units_20_more, DP04_0014PE AS units_mobile, DP04_0015PE AS units_boat_rv_van, DP04_0017PE AS built_2014_later, DP04_0018PE AS built_2010_2013, DP04_0019PE AS built_2000_2009, DP04_0020PE AS built_1990_1999, DP04_0021PE AS built_1980_1989, DP04_0022PE AS built_1970_1979, DP04_0023PE AS built_1960_1969, DP04_0024PE AS built_1950_1959, DP04_0025PE AS built_1940_1949, DP04_0026PE AS built_1939_earlier, DP04_0028PE AS rooms_1, DP04_0029PE AS rooms_2, DP04_0030PE AS rooms_3, DP04_0031PE AS rooms_4, DP04_0032PE AS rooms_5, DP04_0033PE AS rooms_6, DP04_0034PE AS rooms_7, DP04_0035PE AS rooms_8, DP04_0036PE AS rooms_9, DP04_0037E AS rooms_median, DP04_0046PE AS occupied_owner, DP04_0047PE AS occupied_renter, DP04_0081PE AS value_50000_less, DP04_0082PE AS value_50000_99999, DP04_0083PE AS value_100000_149000, DP04_0084PE AS value_150000_199999, DP04_0085PE AS value_200000_299999, DP04_0086PE AS value_300000_499999, DP04_0087PE AS value_500000_999999, DP04_0088PE AS value_1000000_more, DP04_0089E AS value_median, DP04_0127PE AS rent_500_less, DP04_0128PE AS rent_500_999, DP04_0129PE AS rent_1000_1499, DP04_0130PE AS rent_1500_1999, DP04_0131PE AS rent_2000_2499, DP04_0132PE AS rent_2500_2999, DP04_0133PE AS rent_3000_more, DP04_0134E AS rent_median, DP04_0137PE AS grapi_15_less, DP04_0138PE AS grapi_15_19, DP04_0139PE AS grapi_20_24, DP04_0140PE AS grapi_25_29, DP04_0141PE AS grapi_30_34, DP04_0142PE AS grapi_35_more, DP02_0060PE AS edu_9_less, DP02_0061PE AS edu_9_12, DP02_0062PE AS edu_highschool, DP02_0063PE AS edu_college_nodegree, DP02_0064PE AS edu_associate_degree, DP02_0065PE AS edu_bachelor_degree, DP02_0066PE AS edu_grad_degree, DP02_0067PE AS edu_highschool_higher, DP02_0068PE AS edu_bachelor_higher, DP02_0125PE AS ancestry_american, DP02_0126PE AS ancestry_arab, DP02_0127PE AS ancestry_czech, DP02_0128PE AS ancestry_danish, DP02_0129PE AS ancestry_dutch, DP02_0130PE AS ancestry_english, DP02_0131PE AS ancestry_french, DP02_0132PE AS ancestry_frenchcanadian, DP02_0133PE AS ancestry_german, DP02_0134PE AS ancestry_greek, DP02_0135PE AS ancestry_hungarian, DP02_0136PE AS ancestry_irish, DP02_0137PE AS ancestry_italian, DP02_0138PE AS ancestry_lithuanian, DP02_0139PE AS ancestry_norwegian, DP02_0140PE AS ancestry_polish, DP02_0141PE AS ancestry_portuguese, DP02_0142PE AS ancestry_russian, DP02_0143PE AS ancestry_scottish_irish, DP02_0144PE AS ancestry_scottish, DP02_0145PE AS ancestry_slovak, DP02_0146PE AS ancestry_subsaharan_african, DP02_0147PE AS ancestry_swedish, DP02_0148PE AS ancestry_swiss, DP02_0149PE AS ancestry_ukranian, DP02_0150PE AS ancestry_welsh, DP02_0151PE AS ancestry_west_indian FROM place a, dp02_2020 b, dp03_2020 c, dp04_2020 d, dp05_2020 e WHERE a.geoid = SUBSTRING(b.geo_id,10,7) AND a.geoid = SUBSTRING(c.geo_id,10,7) AND a.geoid = SUBSTRING(d.geo_id,10,7) AND a.geoid = SUBSTRING(e.geo_id,10,7);"

# census tracts
psql -d us -c "CREATE TABLE tract2020 AS SELECT a.\"SHAPE\", a.geoid, a.namelsad as name, DP05_0001E AS pop2020, DP05_0005PE AS age_5_under, DP05_0006PE AS age_5_9, DP05_0007PE AS age_10_14, DP05_0008PE AS age_15_19, DP05_0009PE AS age_20_24, DP05_0010PE AS age_25_34, DP05_0011PE AS age_35_44, DP05_0012PE AS age_45_54, DP05_0013PE AS age_55_59, DP05_0014PE AS age_60_64, DP05_0015PE AS age_65_74, DP05_0016PE AS age_75_84, DP05_0017PE AS age_85_over, DP05_0018E AS age_median, DP05_0019PE AS age_under18, DP05_0020PE AS age_16_over, DP05_0021PE AS age_18_over, DP05_0022PE AS age_21_over, DP05_0023PE AS age_62_over, DP05_0024PE AS age_65_over, DP05_0035PE AS race_mixed, DP05_0037PE AS race_white, DP05_0038PE AS race_black, DP05_0039PE AS race_native, DP05_0044PE AS race_asian, DP05_0052PE AS race_pacific, DP05_0071PE AS race_hispanic, DP05_0086E AS housing_total, DP03_0004PE AS employed, DP03_0005PE AS unemployed, DP03_0027PE AS job_bus_sci_art, DP03_0028PE AS job_service, DP03_0029PE AS job_sales, DP03_0030PE AS job_construction, DP03_0031PE AS job_prod_transpo, DP03_0033PE AS in_agri_mining, DP03_0034PE AS in_construction, DP03_0035PE AS in_manufacturing, DP03_0036PE AS in_wholesale, DP03_0037PE AS in_retail, DP03_0038PE AS in_transportation, DP03_0039PE AS in_information, DP03_0040PE AS in_finance, DP03_0041PE AS in_professional, DP03_0042PE AS in_education_health, DP03_0043PE AS in_arts_food, DP03_0044PE AS in_other, DP03_0045PE AS in_public_admin, DP03_0047PE AS class_private, DP03_0048PE AS class_govt, DP03_0049PE AS class_selfemployed, DP03_0050PE AS class_unpaid, DP03_0052PE AS income_10000_less, DP03_0053PE AS income_10000_14999, DP03_0054PE AS income_15000_24999, DP03_0055PE AS income_25000_34999, DP03_0056PE AS income_35000_49999, DP03_0057PE AS income_50000_74999, DP03_0058PE AS income_75000_99999, DP03_0059PE AS income_100000_149999, DP03_0060PE AS income_150000_199999, DP03_0061PE AS income_200000_more, DP03_0062E AS income_median, DP03_0063E AS income_mean, DP03_0088E AS income_percapita, DP03_0092E AS earnings_median, DP03_0093E AS earnings_male, DP03_0094E AS earnings_female, DP04_0001E AS housing_units, DP04_0002PE AS housing_occupied, DP04_0003PE AS housing_vacant, DP04_0004PE AS vacancy_homeowner, DP04_0005PE AS vacancy_rental, DP04_0007PE AS units_1_detached, DP04_0008PE AS units_1_attached, DP04_0009PE AS units_2, DP04_0010PE AS units_3_4, DP04_0011PE AS units_5_9, DP04_0012PE AS units_10_19, DP04_0013PE AS units_20_more, DP04_0014PE AS units_mobile, DP04_0015PE AS units_boat_rv_van, DP04_0017PE AS built_2014_later, DP04_0018PE AS built_2010_2013, DP04_0019PE AS built_2000_2009, DP04_0020PE AS built_1990_1999, DP04_0021PE AS built_1980_1989, DP04_0022PE AS built_1970_1979, DP04_0023PE AS built_1960_1969, DP04_0024PE AS built_1950_1959, DP04_0025PE AS built_1940_1949, DP04_0026PE AS built_1939_earlier, DP04_0028PE AS rooms_1, DP04_0029PE AS rooms_2, DP04_0030PE AS rooms_3, DP04_0031PE AS rooms_4, DP04_0032PE AS rooms_5, DP04_0033PE AS rooms_6, DP04_0034PE AS rooms_7, DP04_0035PE AS rooms_8, DP04_0036PE AS rooms_9, DP04_0037E AS rooms_median, DP04_0046PE AS occupied_owner, DP04_0047PE AS occupied_renter, DP04_0081PE AS value_50000_less, DP04_0082PE AS value_50000_99999, DP04_0083PE AS value_100000_149000, DP04_0084PE AS value_150000_199999, DP04_0085PE AS value_200000_299999, DP04_0086PE AS value_300000_499999, DP04_0087PE AS value_500000_999999, DP04_0088PE AS value_1000000_more, DP04_0089E AS value_median, DP04_0127PE AS rent_500_less, DP04_0128PE AS rent_500_999, DP04_0129PE AS rent_1000_1499, DP04_0130PE AS rent_1500_1999, DP04_0131PE AS rent_2000_2499, DP04_0132PE AS rent_2500_2999, DP04_0133PE AS rent_3000_more, DP04_0134E AS rent_median, DP04_0137PE AS grapi_15_less, DP04_0138PE AS grapi_15_19, DP04_0139PE AS grapi_20_24, DP04_0140PE AS grapi_25_29, DP04_0141PE AS grapi_30_34, DP04_0142PE AS grapi_35_more, DP02_0060PE AS edu_9_less, DP02_0061PE AS edu_9_12, DP02_0062PE AS edu_highschool, DP02_0063PE AS edu_college_nodegree, DP02_0064PE AS edu_associate_degree, DP02_0065PE AS edu_bachelor_degree, DP02_0066PE AS edu_grad_degree, DP02_0067PE AS edu_highschool_higher, DP02_0068PE AS edu_bachelor_higher, DP02_0125PE AS ancestry_american, DP02_0126PE AS ancestry_arab, DP02_0127PE AS ancestry_czech, DP02_0128PE AS ancestry_danish, DP02_0129PE AS ancestry_dutch, DP02_0130PE AS ancestry_english, DP02_0131PE AS ancestry_french, DP02_0132PE AS ancestry_frenchcanadian, DP02_0133PE AS ancestry_german, DP02_0134PE AS ancestry_greek, DP02_0135PE AS ancestry_hungarian, DP02_0136PE AS ancestry_irish, DP02_0137PE AS ancestry_italian, DP02_0138PE AS ancestry_lithuanian, DP02_0139PE AS ancestry_norwegian, DP02_0140PE AS ancestry_polish, DP02_0141PE AS ancestry_portuguese, DP02_0142PE AS ancestry_russian, DP02_0143PE AS ancestry_scottish_irish, DP02_0144PE AS ancestry_scottish, DP02_0145PE AS ancestry_slovak, DP02_0146PE AS ancestry_subsaharan_african, DP02_0147PE AS ancestry_swedish, DP02_0148PE AS ancestry_swiss, DP02_0149PE AS ancestry_ukranian, DP02_0150PE AS ancestry_welsh, DP02_0151PE AS ancestry_west_indian FROM census_tract a, dp02_tract_2020 b, dp03_tract_2020 c, dp04_tract_2020 d, dp05_tract_2020 e WHERE a.geoid = SUBSTRING(b.geo_id,10,11) AND a.geoid = SUBSTRING(c.geo_id,10,11) AND a.geoid = SUBSTRING(d.geo_id,10,11) AND a.geoid = SUBSTRING(e.geo_id,10,11);"

# pumas
# join tracts to puma
psql -d us -c 'ALTER TABLE tract2020 ADD COLUMN puma VARCHAR;'
psql -d us -c 'UPDATE tract2020 a SET puma = b.geoid10 FROM puma b WHERE ST_Intersects(b.geom, ST_Centroid(a."SHAPE"));'

# IMPORTANT! AVERAGE percentages, SUM counts
psql -d us -c "CREATE TABLE puma2020 AS SELECT geom AS \"SHAPE\", geoid10 AS geoid, namelsad10 AS name FROM puma;"
psql -Aqt -d us -c '\d tract2020' | grep -v "SHAPE" | grep -v "geoid" | grep -v "name" | sed -e 's/|.*//g' | while read column; do
  psql -d us -c "ALTER TABLE puma2020 ADD COLUMN ${column} REAL;"
  case ${column} in
    pop2020|housing_total|housing_units)
      psql -d us -c "WITH b AS (SELECT a.puma, SUM(CAST(b.${column} AS REAL))::numeric(10,0) total FROM census_tract a, tract2020 b WHERE CAST(b.${column} AS TEXT) ~ '^[0-9\\\.]+$' AND a.geoid = b.geoid GROUP BY a.puma) UPDATE puma2020 a SET ${column} = b.total FROM b WHERE a.geoid = b.puma;"
    ;;
    *)
      psql -d us -c "WITH b AS (SELECT a.puma, AVG(CAST(b.${column} AS REAL))::numeric(10,0) total FROM census_tract a, tract2020 b WHERE CAST(b.${column} AS TEXT) ~ '^[0-9\\\.]+$' AND a.geoid = b.geoid GROUP BY a.puma) UPDATE puma2020 a SET ${column} = b.total FROM b WHERE a.geoid = b.puma;"
    ;;
  esac
done
```

Add zscores of these columns.
```bash
# states
psql -qAtX -d us -c '\d state2020;' | grep -v "SHAPE" | grep -v "geoid" | grep -v "name" | sed -e 's/|.*//g' | while read column; do
  psql -d us -c "ALTER TABLE state2020 ADD COLUMN zscore_${column} REAL;"
  psql -d us -c "WITH b AS (SELECT geoid, (${column}::real - AVG(${column}::real) OVER()) / STDDEV(${column}::real) OVER() AS zscore FROM state2020 WHERE CAST(${column} AS TEXT) ~ '^[0-9\\\.]+$') UPDATE state2020 a SET zscore_${column} = b.zscore FROM b WHERE a.geoid = b.geoid;"
done

# counties
psql -qAtX -d us -c '\d county2020;' | grep -v "SHAPE" | grep -v "geoid" | grep -v "name" | sed -e 's/|.*//g' | while read column; do
  psql -d us -c "ALTER TABLE county2020 ADD COLUMN zscore_${column} REAL;"
  psql -d us -c "WITH b AS (SELECT geoid, (${column}::real - AVG(${column}::real) OVER()) / STDDEV(${column}::real) OVER() AS zscore FROM county2020 WHERE CAST(${column} AS TEXT) ~ '^[0-9\\\.]+$') UPDATE county2020 a SET zscore_${column} = b.zscore FROM b WHERE a.geoid = b.geoid;"
done

# places
psql -qAtX -d us -c '\d place2020;' |  grep -v "SHAPE" | grep -v "geoid" | grep -v "name" | sed -e 's/|.*//g' | while read column; do
  psql -d us -c "ALTER TABLE place2020 ADD COLUMN zscore_${column} REAL;"
  psql -d us -c "WITH b AS (SELECT geoid, (${column}::real - AVG(${column}::real) OVER()) / STDDEV(${column}::real) OVER() AS zscore FROM place2020 WHERE ${column}::text ~ '^[0-9\\\.]+$') UPDATE place2020 a SET zscore_${column} = b.zscore FROM b WHERE a.geoid = b.geoid;"
done

# pumas
psql -qAtX -d us -c '\d puma2020' | grep -v "SHAPE" | grep -v "geoid" | grep -v "name" | sed -e 's/|.*//g' | while read column; do
  psql -d us -c "ALTER TABLE puma2020 ADD COLUMN zscore_${column} REAL;"
  psql -d us -c "WITH b AS (SELECT geoid, (${column}::real - AVG(${column}::real) OVER()) / STDDEV(${column}::real) OVER() AS zscore FROM puma2020 WHERE CAST(${column} AS TEXT) ~ '^[0-9\\\.]+$') UPDATE puma2020 a SET zscore_${column} = b.zscore FROM b WHERE a.geoid = b.geoid;"
done
```

Find zscores over 1.65 and compare those variables at different geographies.
```bash
# state
psql -d us -c "ALTER TABLE state2020 ADD COLUMN zscore_1_65 jsonb;"
psql -Aqt -d us -c "COPY (SELECT geoid, name from state2020) TO STDOUT DELIMITER E'\t';" | while IFS=$'\t' read -a array; do
  columns=$(psql -Aqt -d us -c "WITH b AS (SELECT $(psql -Aqt -d us -c '\d state2020' | grep "zscore_" | sed -e 's/|.*//g' | paste -sd,) FROM state2020 WHERE geoid = '${array[0]}') SELECT (x).key FROM (SELECT EACH(hstore(b)) x FROM b) q WHERE CAST((x).value AS VARCHAR) ~ '^[0-9\\\.]+$' AND ABS(CAST((x).value AS REAL)) >= 1.65;" | paste -sd,)
  psql -d us -c "UPDATE state2020 a SET zscore_1_65 = (SELECT to_jsonb(inputs) FROM (SELECT $(echo ${columns} | tr ',' '\n' | sed -e 's/zscore_//g' -e "s/.*/CONCAT\('{state:', a\.\0, '|us:', b\.\0\, '}') AS \0/g" | paste -sd,) FROM state2020 a, us2020 b WHERE a.geoid = '${array[0]}') inputs) WHERE a.geoid = '${array[0]}';"
done

# county
psql -d us -c "ALTER TABLE county2020 ADD COLUMN zscore_1_65 jsonb;"
psql -Aqt -d us -c "COPY (SELECT geoid, name from county2020) TO STDOUT DELIMITER E'\t';" | while IFS=$'\t' read -a array; do
  columns=$(psql -Aqt -d us -c "WITH b AS (SELECT $(psql -Aqt -d us -c '\d county2020' | grep "zscore_" | sed -e 's/|.*//g' | paste -sd,) FROM county2020 WHERE geoid = '${array[0]}') SELECT (x).key FROM (SELECT EACH(hstore(b)) x FROM b) q WHERE CAST((x).value AS VARCHAR) ~ '^[0-9\\\.]+$' AND ABS(CAST((x).value AS REAL)) >= 1.65;" | paste -sd,)
  psql -d us -c "UPDATE county2020 a SET zscore_1_65 = (SELECT to_jsonb(inputs) FROM (SELECT $(echo ${columns} | tr ',' '\n' | sed -e 's/zscore_//g' -e "s/.*/CONCAT\('{county:', a\.\0, '|state:', b\.\0\, '|us:', c\.\0\, '}') AS \0/g" | paste -sd,) FROM county2020 a, state2020 b, us2020 c WHERE a.geoid = '${array[0]}' AND SUBSTRING(a.geoid,1,2) = b.geoid) inputs) WHERE a.geoid = '${array[0]}';"
done

# place
psql -d us -c "ALTER TABLE place2020 ADD COLUMN zscore_1_65 jsonb;"
psql -Aqt -d us -c "COPY (SELECT geoid, name from place2020) TO STDOUT DELIMITER E'\t';" | while IFS=$'\t' read -a array; do
  columns=$(psql -Aqt -d us -c "WITH b AS (SELECT $(psql -Aqt -d us -c '\d place2020' | grep "zscore_" | sed -e 's/|.*//g' | paste -sd,) FROM place2020 WHERE geoid = '${array[0]}') SELECT (x).key FROM (SELECT EACH(hstore(b)) x FROM b) q WHERE CAST((x).value AS VARCHAR) ~ '^[0-9\\\.]+$' AND ABS(CAST((x).value AS REAL)) >= 1.65;" | paste -sd,)
  psql -d us -c "UPDATE place2020 a SET zscore_1_65 = (SELECT to_jsonb(inputs) FROM (SELECT $(echo ${columns} | tr ',' '\n' | sed -e 's/zscore_//g' -e "s/.*/CONCAT\('{place:', a\.\0, '|state:', b\.\0\, '|us:', c\.\0\, '}') AS \0/g" | paste -sd,) FROM place2020 a, state2020 b, us2020 c WHERE a.geoid = '${array[0]}' AND SUBSTRING(a.geoid,1,2) = b.geoid) inputs) WHERE a.geoid = '${array[0]}';"
done

# puma
psql -d us -c "ALTER TABLE puma2020 ADD COLUMN zscore_1_65 jsonb;"
psql -Aqt -d us -c "COPY (SELECT a.geoid, a.name, SUBSTRING(b.geoid,1,5) from puma2020 a, census_tract b WHERE a.geoid = b.puma) TO STDOUT DELIMITER E'\t';" | while IFS=$'\t' read -a array; do
  columns=$(psql -Aqt -d us -c "WITH b AS (SELECT $(psql -Aqt -d us -c '\d puma2020' | grep "zscore_" | sed -e 's/|.*//g' | paste -sd,) FROM puma2020 WHERE geoid = '${array[0]}') SELECT (x).key FROM (SELECT EACH(hstore(b)) x FROM b) q WHERE CAST((x).value AS VARCHAR) ~ '^[0-9\\\.]+$' AND ABS(CAST((x).value AS REAL)) >= 1.65;" | paste -sd,)
  psql -d us -c "UPDATE puma2020 a SET zscore_1_65 = (SELECT to_jsonb(inputs) FROM (SELECT $(echo ${columns} | tr ',' '\n' | sed -e 's/zscore_//g' -e "s/.*/CONCAT\('{puma:', a\.\0, '|county:', b\.\0\, '|state:', c\.\0\, '|us:', d\.\0\, '}') AS \0/g" | paste -sd,) FROM puma2020 a, county2020 b, state2020 c, us2020 d WHERE a.geoid = '${array[0]}' AND b.geoid = '${array[2]}' AND SUBSTRING(a.geoid,1,2) = c.geoid) inputs) WHERE a.geoid = '${array[0]}';"
done
```

Add census geoid to geonames
```bash
# block
psql -d us -c 'ALTER TABLE geonames_us ADD COLUMN geoid_block VARCHAR;'
psql -d us -c 'UPDATE geonames_us a SET geoid_block = b.geoid FROM block20 b WHERE ST_Intersects(a.geom, b."SHAPE");'

# place
psql -d us -c "ALTER TABLE geonames_us ADD COLUMN geoid_place varchar;"
psql -d us -c "UPDATE geonames_us a SET geoid_place = b.geoid FROM place b WHERE SUBSTRING(a.geoid_block,1,2) = SUBSTRING(b.geoid,1,2) AND ST_Intersects(a.geom, b.\"SHAPE\");"

# puma
psql -d us -c "ALTER TABLE geonames_us ADD COLUMN geoid_puma varchar;"
psql -d us -c "UPDATE geonames_us a SET geoid_puma = b.puma FROM census_tract b WHERE SUBSTRING(a.geoid_block,1,11) = b.geoid;"
```

Create table of osm points by state (easier to work with).
```bash
psql -Aqt -d us -c "COPY (SELECT geoid from state2020) TO STDOUT DELIMITER E'\t';" | while IFS=$'\t' read -a array; do
  psql -d us -c "CREATE TABLE points_${array[0]} AS SELECT * FROM points_us WHERE SUBSTRING(geoid_block,1,2) = '${array[0]}';"
done
```

Add census geoid to osm points.
```bash
# block
psql -Aqt -d us -c "COPY (SELECT geoid from state2020) TO STDOUT DELIMITER E'\t';" | while read geoid; do
  psql -d us -c 'ALTER TABLE points_${geoid} ADD COLUMN geoid_block VARCHAR;'
  psql -d us -c 'UPDATE points_${geoid} a SET geoid_block = b.geoid FROM block20 b WHERE ST_Intersects(a.wkb_geometry, b."SHAPE");'
done

# place
psql -Aqt -d us -c "COPY (SELECT geoid from state2020) TO STDOUT DELIMITER E'\t';" | while read geoid; do
  psql -d us -c "ALTER TABLE points_${geoid} ADD COLUMN geoid_place varchar;"
  psql -d us -c "UPDATE points_${geoid} a SET geoid_place = b.geoid FROM place b WHERE SUBSTRING(a.geoid_block,1,2) = SUBSTRING(b.geoid,1,2) AND ST_Intersects(a.wkb_geometry, b.\"SHAPE\");"
done

# puma
psql -Aqt -d us -c "COPY (SELECT geoid from state2020) TO STDOUT DELIMITER E'\t';" | while read geoid; do
  psql -d us -c "ALTER TABLE points_${geoid} ADD COLUMN geoid_puma varchar;"
  psql -d us -c "UPDATE points_${geoid} a SET geoid_puma = b.puma FROM census_tract b WHERE SUBSTRING(a.geoid_block,1,11) = b.geoid;"
done
```

Add brand names, counts by geography
```bash
# state
psql -d us -c "ALTER TABLE state2020 ADD COLUMN brand json;"
psql -Aqt -d us -c "COPY (SELECT geoid from state2020) TO STDOUT DELIMITER E'\t';" | while read geoid; do
  psql -d us -c "UPDATE state2020 a SET brand = (SELECT json_agg(json_build_object(value::text, count::text)) FROM (SELECT value, COUNT(value) count FROM (SELECT other_tags->'brand' value FROM points_${geoid}) stat WHERE value IS NOT NULL GROUP BY value ORDER BY count DESC) stats) WHERE a.geoid = '${geoid}';"
done

# counties
psql -d us -c "ALTER TABLE county2020 ADD COLUMN brand json;"
psql -Aqt -d us -c "COPY (SELECT geoid from county2020) TO STDOUT DELIMITER E'\t';" | while read geoid; do
  psql -d us -c "UPDATE county2020 a SET brand = (SELECT json_agg(json_build_object(value::text, count::text)) FROM (SELECT value, COUNT(value) count FROM (SELECT geoid_block, other_tags->'brand' value FROM points_${geoid:0:2}) stat WHERE value IS NOT NULL AND SUBSTRING(stat.geoid_block,1,5) = '${geoid}' GROUP BY value ORDER BY count DESC) stats) WHERE a.geoid = '${geoid}';"
done

# places
psql -d us -c "ALTER TABLE place2020 ADD COLUMN brand json;"
psql -Aqt -d us -c "COPY (SELECT geoid from place2020) TO STDOUT DELIMITER E'\t';" | while read geoid; do
  psql -d us -c "UPDATE place2020 a SET brand = (SELECT json_agg(json_build_object(value::text, count::text)) FROM (SELECT value, COUNT(value) count FROM (SELECT wkb_geometry, other_tags->'brand' value FROM points_${geoid:0:2}) stat WHERE value IS NOT NULL AND a.geoid_place = '${geoid}' GROUP BY value ORDER BY count DESC) stats) WHERE a.geoid = '${geoid}';"
done

# pumas
psql -d us -c "ALTER TABLE puma2020 ADD COLUMN brand json;"
psql -Aqt -d us -c "COPY (SELECT geoid from puma2020) TO STDOUT DELIMITER E'\t';" | while read geoid; do
  psql -d us -c "UPDATE puma2020 a SET brand = (SELECT json_agg(json_build_object(value::text, count::text)) FROM (SELECT value, COUNT(value) count FROM (SELECT wkb_geometry, other_tags->'brand' value FROM points_${geoid:0:2}) stat WHERE value IS NOT NULL AND a.geoid_puma = '${geoid}' GROUP BY value ORDER BY count DESC) stats) WHERE a.geoid = '${geoid}';"
done
```

## 3. Exporting

Export geojson files.
```bash
# county2020 quartiles (with simplified geometry from naturalearth)
column=pop2020
psql -d us -c "COPY (SELECT jsonb_build_object('type', 'FeatureCollection', 'features', jsonb_agg(feature)) FROM (SELECT jsonb_build_object('type', 'Feature', 'id', quartile, 'geometry', ST_AsGeoJSON(ST_Transform(geom,4326))::jsonb, 'properties', to_jsonb(inputs) - 'geom') AS feature FROM (WITH stats AS (SELECT DISTINCT PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY ${column}::real) q1, PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY ${column}::real) q2, PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY ${column}::real) q3 FROM county2020) SELECT 'q1' AS quartile, MIN(${column}::real) AS min, MAX(${column}::real) AS max, ST_Union(ST_Buffer(b.geom,0)) AS geom FROM county2020 a, ne_10m_admin_2_counties b, stats WHERE a.${column}::real < stats.q1 AND a.geoid = b.code_local UNION ALL SELECT 'q2' AS quartile, MIN(${column}::real) AS min, MAX(${column}::real) AS max, ST_Union(ST_Buffer(b.geom,0)) AS geom FROM county2020 a, ne_10m_admin_2_counties b, stats WHERE a.${column}::real >= stats.q1 AND a.${column}::real < stats.q2 AND a.geoid = b.code_local UNION ALL SELECT 'q3' AS quartile, MIN(${column}::real) AS min, MAX(${column}::real) AS max, ST_Union(ST_Buffer(b.geom,0)) AS geom FROM county2020 a, ne_10m_admin_2_counties b, stats WHERE a.${column}::real >= stats.q2 AND a.${column}::real < stats.q3 AND a.geoid = b.code_local UNION ALL SELECT 'q4' AS quartile, MIN(${column}::real) AS min, MAX(${column}::real) AS max, ST_Union(ST_Buffer(b.geom,0)) AS geom FROM county2020 a, ne_10m_admin_2_counties b, stats WHERE a.${column}::real >= stats.q3 AND a.geoid = b.code_local) inputs) features) TO STDOUT;" > data/county_${column}_quartile.geojson

# co_est2021 quartiles (with simplified geometry from naturalearth)
for column in popestimate2021 npopchg2021 naturalchg2021 netmig2021; do
  psql -d us -c "COPY (SELECT jsonb_build_object('type', 'FeatureCollection', 'features', jsonb_agg(feature)) FROM (SELECT jsonb_build_object('type', 'Feature', 'id', quartile, 'geometry', ST_AsGeoJSON(ST_Transform(geom,4326))::jsonb, 'properties', to_jsonb(inputs) - 'geom') AS feature FROM (WITH stats AS (SELECT DISTINCT PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY ${column}::real) q1, PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY ${column}::real) q2, PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY ${column}::real) q3 FROM co_est2021) SELECT 'q1' AS quartile, MIN(${column}::real) AS min, MAX(${column}::real) AS max, ST_Union(ST_Buffer(b.geom,0)) AS geom FROM co_est2021 a, ne_10m_admin_2_counties_lakes b, stats WHERE a.${column}::real < stats.q1 AND a.state_county = b.code_local UNION ALL SELECT 'q2' AS quartile, MIN(${column}::real) AS min, MAX(${column}::real) AS max, ST_Union(ST_Buffer(b.geom,0)) AS geom FROM co_est2021 a, ne_10m_admin_2_counties_lakes b, stats WHERE a.${column}::real >= stats.q1 AND a.${column}::real < stats.q2 AND a.state_county = b.code_local UNION ALL SELECT 'q3' AS quartile, MIN(${column}::real) AS min, MAX(${column}::real) AS max, ST_Union(ST_Buffer(b.geom,0)) AS geom FROM co_est2021 a, ne_10m_admin_2_counties_lakes b, stats WHERE a.${column}::real >= stats.q2 AND a.${column}::real < stats.q3 AND a.state_county = b.code_local UNION ALL SELECT 'q4' AS quartile, MIN(${column}::real) AS min, MAX(${column}::real) AS max, ST_Union(ST_Buffer(b.geom,0)) AS geom FROM co_est2021 a, ne_10m_admin_2_counties_lakes b, stats WHERE a.${column}::real >= stats.q3 AND a.state_county = b.code_local) inputs) features) TO STDOUT;" > data/geojson/county_${column}_quartile.geojson
done

# top 10 counties ranked by column
for column in popestimate2021 npopchg2021 naturalchg2021 netmig2021; do
  psql -d us -c "COPY (SELECT jsonb_build_object('type', 'FeatureCollection', 'features', jsonb_agg(feature)) FROM (SELECT jsonb_build_object('type', 'Feature', 'id', state_county, 'geometry', ST_AsGeoJSON(ST_Transform(geom,4326))::jsonb, 'properties', to_jsonb(inputs) - 'geom' - 'state_county') AS feature FROM (SELECT b.geom, a.ctyname, a.state_county, DENSE_RANK() OVER (ORDER BY a.${column}::real DESC) rank, TO_CHAR(a.popestimate2021::int, 'FM9,999,999,999') AS popestimate2021, TO_CHAR(a.npopchg2021::int, 'FM9,999,999,999') AS npopchg2021, TO_CHAR(a.naturalchg2021::int, 'FM9,999,999,999') AS naturalchg2021, TO_CHAR(a.netmig2021::numeric, 'FM9,999,999,999') AS netmig2021, ROUND(a.rnaturalchg2021::numeric, 2) AS rnaturalchg2021, ROUND(a.rnetmig2021::numeric, 2) AS rnetmig2021 FROM co_est2021 a, ne_10m_admin_2_counties_lakes b WHERE a.sumlev = '050' AND a.state_county = b.code_local ORDER BY ${column}::real DESC LIMIT 10) inputs) features) TO STDOUT;" > data/geojson/county_${column}_top_10.geojson
done

```
