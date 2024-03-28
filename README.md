# American Geography

Updated for American Community Survey 5-Year 2022 released Dec 2023.

1. [Processing](#importing)
2. [Z-scores](#z-scores)
3. [Exporting](#exporting)
4. [References](#references)

## Processing

Census geography files  
```shell
#download national, substate files from https://www2.census.gov/geo/tiger/TGRGDB23/
#download puma with: curl https://www2.census.gov/geo/tiger/TIGER2023/PUMA/ | grep 'tl_2022' | sed -e 's/^.*href="/https:\/\/www2\.census\.gov\/geo\/tiger\/TIGER2023\/PUMA\//g' -e 's/".*$//g' | while read file; do wget ${file}; sleep $[ ( $RANDOM % 10 )  + 1 ]s; done

# national level
ogr2ogr -overwrite -skipfailures --config PG_USE_COPY YES -f PGDump -t_srs "EPSG:3857" /vsistdout/ tlgdb_2023_a_us_nationgeo.gdb.zip | psql -d us -f -

# substate level
ogr2ogr -overwrite -skipfailures --config PG_USE_COPY YES -f PGDump -t_srs "EPSG:3857" /vsistdout/ tlgdb_2023_a_us_substategeo.gdb.zip | psql -d us -f -

# union census designated places and incorporated places
psql -d us -c "DROP TABLE IF EXISTS place; CREATE TABLE place AS (SELECT * FROM incorporated_place UNION ALL SELECT * FROM census_designated_place);"
# add county geoid
ALTER TABLE place ADD COLUMN county_geoid VARCHAR;
UPDATE place a SET county_geoid = (SELECT b.geoid FROM county b WHERE ST_Intersects(a.shape, b.shape) ORDER BY ST_Area(ST_Intersection(a.shape, b.shape)) DESC LIMIT 1);

# merge state puma files then import
ogrmerge.py -overwrite_ds -single -nln puma -o puma.gpkg $(ls *.zip | sed 's/^/\/vsizip\//g' | paste -sd' ')
ogr2ogr -overwrite -skipfailures -nlt promote_to_multi --config PG_USE_COPY YES -f PGDump -t_srs "EPSG:3857" /vsistdout/ puma.gpkg | psql -d us -f -

# blocks
ogr2ogr -overwrite -skipfailures --config PG_USE_COPY YES -f PGDump -t_srs "EPSG:3857" /vsistdout/ tlgdb_2023_a_us_block.gdb.zip | psql -d us -f -
```

Census population change, components of change...  
```shell
#https://www2.census.gov/programs-surveys/popest/datasets/

# clean up and import
geographies=('cbsa' 'co' 'csa' 'nst')
for geography in ${geographies[*]}; do
  iconv -f latin1 -t ascii//TRANSLIT ${geography}'-est2023-alldata.csv' > ${geography}'-est2023-alldata-iconv.csv'
  psql -d us -c "DROP TABLE IF EXISTS ${geography}_est2023; CREATE TABLE ${geography}_est2023($(head -1 ${geography}-est2023-alldata-iconv.csv | sed -e 's/,/ VARCHAR,/g' -e 's/$/ VARCHAR/g'));"
  psql -d us -c "\COPY ${geography}_est2023 FROM ${geography}-est2023-alldata-iconv.csv WITH CSV HEADER;"
done

# add missing geoid column
ALTER TABLE co_est2023 ADD COLUMN geoid varchar;
UPDATE co_est2023 SET geoid = CONCAT(state, county)::VARCHAR;
```

Blocks  
```shell
# download by state from: https://data.census.gov/table?g=040XX00US01$1000000

files=('P1' 'H1')
geography='block'
for file in ${files[*]}; do
  table=${file}_${geography}2020
  cat ${geography}/DECENNIALDHC2020.${file}-Data.csv | awk 'NR!=2' | iconv -f latin1 -t ascii//TRANSLIT | sed -e 's/,$//g' -e 's/i>>?//g' > ${geography}/DECENNIALDHC2020.${file}-Data_iconv.csv
  psql -d us -c "DROP TABLE IF EXISTS ${table}; CREATE TABLE ${table}($(head -1 ${geography}/DECENNIALDHC2020.${file}-Data_iconv.csv | sed -e 's/"//g' -e 's/,/ VARCHAR,/g' -e 's/$/ VARCHAR/g'));"
  psql -d us -c "\COPY ${table} FROM ${geography}/DECENNIALDHC2020.${file}-Data_iconv.csv WITH CSV HEADER;"
done

# add housing, population totals
ALTER TABLE block20 ADD COLUMN h1 VARCHAR; ALTER TABLE block20 ADD COLUMN p1 VARCHAR;
UPDATE block20 a SET h1 = b.h1_001n FROM h1_block2020 b WHERE a.geoid20 = split_part(b.geo_id, 'US', 2);
UPDATE block20 a SET p1 = b.p1_001n FROM p1_block2020 b WHERE a.geoid20 = split_part(b.geo_id, 'US', 2);
```

Data profiles  
```shell
#https://data.census.gov/table?d=ACS+5-Year+Estimates+Data+Profiles&y=2022&q=DP02,DP03,DP04,DP05&g=010XX00US
#https://data.census.gov/table?d=ACS+5-Year+Estimates+Data+Profiles&y=2022&q=DP02,DP03,DP04,DP05&g=010XX00US$0400000
#https://data.census.gov/table?d=ACS+5-Year+Estimates+Data+Profiles&y=2022&q=DP02,DP03,DP04,DP05&g=010XX00US$0500000
#https://data.census.gov/table?d=ACS+5-Year+Estimates+Data+Profiles&y=2022&q=DP02,DP03,DP04,DP05&g=010XX00US$1400000
#https://data.census.gov/table?d=ACS+5-Year+Estimates+Data+Profiles&y=2022&q=DP02,DP03,DP04,DP05&g=010XX00US$1600000
#https://data.census.gov/table?d=ACS+5-Year+Estimates+Data+Profiles&y=2022&q=DP02,DP03,DP04,DP05&g=010XX00US$7950000

geographies=('us' 'state' 'county' 'place' 'tract' 'puma')
files=('DP02' 'DP03' 'DP04' 'DP05')
for geography in ${geographies[*]}; do
  for file in ${files[*]}; do
    table=${file}_${geography}2022
    cat ${geography}/ACSDP5Y2022.${file}-Data.csv | awk 'NR!=2' | iconv -f latin1 -t ascii//TRANSLIT | sed -e 's/,$//g' > ${geography}/ACSDP5Y2022.${file}-Data_iconv.csv
    psql -d us -c "DROP TABLE IF EXISTS ${table}; CREATE TABLE ${table}($(head -1 ${geography}/ACSDP5Y2022.${file}-Data_iconv.csv | sed -e 's/"//g' -e 's/,/ VARCHAR,/g' -e 's/$/ VARCHAR/g'));"
    psql -d us -c "\COPY ${table} FROM ${geography}/ACSDP5Y2022.${file}-Data_iconv.csv WITH CSV HEADER;"
  done
done
```

Metadata (labels)  
```shell
geographies=('us' 'state' 'county' 'place' 'tract' 'puma')
files=('DP02' 'DP03' 'DP04' 'DP05')
for geography in ${geographies[*]}; do
  for file in ${files[*]}; do
    cat ~/maps/us/dataprofiles/${geography}/ACSDP5Y2022.${file}-Column-Metadata.csv | tail -n +4 | sed -e 's/","/\t/g' -e 's/"//g' > ~/maps/us/dataprofiles/${geography}/ACSDP5Y2022.${file}-Column-Metadata.tsv
    psql -d us -c "DROP TABLE IF EXISTS ${file}_${geography}_metadata; CREATE TABLE ${file}_${geography}_metadata(code VARCHAR, label VARCHAR);"
    psql -d us -c "COPY ${file}_${geography}_metadata FROM ~/maps/us/dataprofiles/${geography}/ACSDP5Y2022.${file}-Column-Metadata.tsv WITH DELIMITER E'\t';"
  done
done
```

Natural Earth  
```shell
# states
ogr2ogr -overwrite -skipfailures --config PG_USE_COPY YES -f PGDump -t_srs "EPSG:3857" -nlt promote_to_multi /vsistdout/ natural_earth_vector_3857.gpkg ne_10m_admin_1_states_provinces_lakes | psql -d us -f -

# counties
ogr2ogr -overwrite -skipfailures --config PG_USE_COPY YES -f PGDump -t_srs "EPSG:3857" -nlt promote_to_multi /vsistdout/ natural_earth_vector_3857.gpkg ne_10m_admin_2_counties_lakes | psql -d us -f -
```

OpenStreetMap 
```shell
# copy tables from osm database
ogr2ogr -overwrite --config PG_USE_COPY YES -f PGDump /vsistdout/ PG:dbname=us us2022 | psql -d osm -f -
ogr2ogr -overwrite --config PG_USE_COPY YES -f PGDump /vsistdout/ PG:dbname=us state2022 | psql -d osm -f -
ogr2ogr -overwrite --config PG_USE_COPY YES -f PGDump /vsistdout/ PG:dbname=us county2022 | psql -d osm -f -
ogr2ogr -overwrite --config PG_USE_COPY YES -f PGDump /vsistdout/ PG:dbname=us place2022 | psql -d osm -f -
ogr2ogr -overwrite --config PG_USE_COPY YES -f PGDump /vsistdout/ PG:dbname=us puma2022 | psql -d osm -f -
```

## Z-scores

Add zscores for percentage columns.  
```shell
#geographies=('state' 'county' 'place' 'tract' 'puma') # do all
geographies=('county')
files=('dp02' 'dp03' 'dp04' 'dp05')
for geography in ${geographies[*]}; do
  for file in ${files[*]}; do
    table=${file}_${geography}2022
    psql -Aqt -d us -c "SELECT column_name FROM information_schema.columns WHERE table_name = '${table}';" | grep ".*pe$" | while read column; do
      psql -d us -c "ALTER TABLE ${table} DROP COLUMN IF EXISTS zscore_${column};"
      psql -d us -c "ALTER TABLE ${table} ADD COLUMN zscore_${column} REAL; WITH b AS (SELECT geo_id, (${column}::real - AVG(${column}::real) OVER()) / STDDEV(${column}::real) OVER() AS zscore FROM ${table} WHERE CAST(${column} AS TEXT) ~ '^[0-9\\\.]+$') UPDATE ${table} a SET zscore_${column} = b.zscore FROM b WHERE a.geo_id = b.geo_id;"
    done
  done
done
```

Find zscores over 1.65.  
```shell
geographies=('county')
files=('dp02' 'dp03' 'dp04' 'dp05')
for geography in ${geographies[*]}; do
  for file in ${files[*]}; do
    table=${file}_${geography}2022
    psql -Aqt -d us -c "COPY (SELECT geo_id, name from ${table}) TO STDOUT DELIMITER E'\t';" | while IFS=$'\t' read -a array; do
      columns=$(psql -Aqt -d us -c "WITH b AS (SELECT $(psql -Aqt -d us -c '\d ${table}' | grep "zscore_" | sed -e 's/|.*//g' | paste -sd,) FROM ${table} WHERE geo_id = '${array[0]}') SELECT (x).key FROM (SELECT EACH(hstore(b)) x FROM b) q WHERE CAST((x).value AS VARCHAR) ~ '^[0-9\\\.]+$' AND ABS(CAST((x).value AS REAL)) >= 1.65;" | paste -sd,)
      psql -d us -c "SELECT to_jsonb(inputs) FROM (SELECT $(echo ${columns} | tr ',' '\n' | sed -e 's/zscore_//g' -e "s/.*/CONCAT\('{county:', a\.\0, '|state:', b\.\0\, '|us:', c\.\0\, '}') AS \0/g" | paste -sd,) FROM ${table} a, state b, us c WHERE a.geo_id = '${array[0]}' AND SUBSTRING(a.geo_id,10,2) = SUBSTRING(b.geo_id,10,2)) inputs;"
    done
  done
done
```

## Tables

Population change in counties (co_est2023)
```sql
# order by change percentage
WITH stats AS (SELECT geoid, ARRAY_AGG(name ORDER BY pop::numeric DESC) AS places FROM (SELECT b.geoid, a.name, a.pop, ROW_NUMBER() OVER (PARTITION BY b.geoid ORDER BY a.pop::numeric DESC) AS row_num FROM place2022 a JOIN county b ON a.county_geoid = b.geoid) subquery WHERE row_num <= 3 GROUP BY geoid) SELECT a.stname, a.ctyname, stats.places, a.popestimate2023, a.npopchg2023, ROUND(((a.popestimate2023::numeric - a.popestimate2022::numeric) / a.popestimate2022::numeric) * 100,2) AS rpopchg2023, RANK() OVER (PARTITION BY a.state ORDER BY ((a.popestimate2023::numeric - a.popestimate2022::numeric) / a.popestimate2022::numeric) * 100 DESC) AS cty_rank, ROUND(((b.popestimate2023::numeric - b.popestimate2022::numeric) / b.popestimate2022::numeric) * 100,2) AS st_rpopchg2023, DENSE_RANK() OVER (ORDER BY ((b.popestimate2023::numeric - b.popestimate2022::numeric) / b.popestimate2022::numeric) * 100 DESC) AS st_rank FROM co_est2023 a JOIN nst_est2023 b ON a.state = b.state JOIN stats ON a.geoid = stats.geoid WHERE a.sumlev = '050' AND b.sumlev = '040' ORDER BY ((a.popestimate2023::numeric - a.popestimate2022::numeric) / a.popestimate2022::numeric) * 100 DESC;

# order by popestimate2023
WITH stats AS (SELECT geoid, ARRAY_AGG(name ORDER BY pop::numeric DESC) AS places FROM (SELECT b.geoid, a.name, a.pop, ROW_NUMBER() OVER (PARTITION BY b.geoid ORDER BY a.pop::numeric DESC) AS row_num FROM place2022 a JOIN county b ON a.county_geoid = b.geoid) subquery WHERE row_num <= 3 GROUP BY geoid) SELECT a.stname, a.ctyname, stats.places, a.popestimate2023, a.npopchg2023, ROUND(((a.popestimate2023::numeric - a.popestimate2022::numeric) / a.popestimate2022::numeric) * 100,2) AS rpopchg2023, RANK() OVER (PARTITION BY a.state ORDER BY ((a.popestimate2023::numeric - a.popestimate2022::numeric) / a.popestimate2022::numeric) * 100 DESC) AS cty_rank, ROUND(((b.popestimate2023::numeric - b.popestimate2022::numeric) / b.popestimate2022::numeric) * 100,2) AS st_rpopchg2023, DENSE_RANK() OVER (ORDER BY ((b.popestimate2023::numeric - b.popestimate2022::numeric) / b.popestimate2022::numeric) * 100 DESC) AS st_rank FROM co_est2023 a JOIN nst_est2023 b ON a.state = b.state JOIN stats ON a.geoid = stats.geoid WHERE a.sumlev = '050' AND b.sumlev = '040' ORDER BY a.popestimate2022::numeric DESC;
```

Export tables  
```shell
# csv
psql -d us -c "COPY (SELECT * FROM (SELECT DENSE_RANK() OVER (ORDER BY a.pop::int DESC) rank, a.name, b.name AS state, a.pop, a.zscore_1_65 FROM place2022 a, state2022 b WHERE SUBSTRING(a.geoid,1,2) = b.geoid) stats WHERE rank <= 100) TO STDOUT DELIMITER E'\t' CSV HEADER;" > tables/place2022_zscore.csv

# html
psql --html -d us -c "SELECT * FROM (SELECT DENSE_RANK() OVER (ORDER BY a.pop::int DESC) rank, a.name, b.name AS state, a.pop, a.zscore_1_65 FROM place2022 a, state2022 b WHERE SUBSTRING(a.geoid,1,2) = b.geoid) stats WHERE rank <= 100;" > tables/place2022_zscore.html

# markdown
psql -d us -c "SELECT * FROM (SELECT DENSE_RANK() OVER (ORDER BY a.pop::int DESC) rank, a.name, b.name AS state, a.pop, a.zscore_1_65 FROM place2022 a, state2022 b WHERE SUBSTRING(a.geoid,1,2) = b.geoid) stats WHERE rank <= 100;" | sed -e 's/-+-/-\|-/g' -e 's/^/\|/g' -e 's/$/\|/g' -e "s/||//g" | grep -v 'rows)|' > tables/place2022_zscore.md
```

## Exporting

Export labels to json.  
```shell
# labels (single file)
psql -Aqt -d us -c "SELECT jsonb_agg(jsonb_build_object(code, label)) FROM dp02_us_metadata WHERE label NOT LIKE '%Margin of Error%' UNION ALL SELECT jsonb_agg(jsonb_build_object(code, label)) FROM dp03_us_metadata WHERE label NOT LIKE '%Margin of Error%' UNION ALL SELECT jsonb_agg(jsonb_build_object(code, label)) FROM dp04_us_metadata WHERE label NOT LIKE '%Margin of Error%' UNION ALL SELECT jsonb_agg(jsonb_build_object(code, label)) FROM dp05_us_metadata WHERE label NOT LIKE '%Margin of Error%';" | tr -d '\n' | sed -e 's/\]\[/, /g' > ~/american-geography/geojson/us_labels.json

# labels (separate files)
files=('dp02' 'dp03' 'dp04' 'dp05')
for file in ${files[*]}; do
  psql -Aqt -d us -c "SELECT jsonb_agg(jsonb_build_object(code, label)) FROM ${file}_us_metadata WHERE label NOT LIKE '%Margin of Error%';" > ~/american-geography/json/${file}_labels.json
done
```

Export geographies to json.  
```shell
# us
files=('dp02' 'dp03' 'dp04' 'dp05')
for file in ${files[*]}; do
  columns=$(psql -Aqt -d us -c "SELECT * FROM ${file}_us_metadata" | grep -v ".*M|" | sed -e 's/|.*//g' | paste -sd,)
  psql -Aqt -d us -c 'SELECT jsonb_agg(row_to_json(fields)) FROM (SELECT name, '"${columns}"' FROM '"${file}"'_us2022) fields;' > ~/american-geography/json/us/${file}_us.json
done 

# states without puerto rico (single file)
files=('dp02' 'dp03' 'dp04' 'dp05')
for file in ${files[*]}; do
  columns=$(psql -Aqt -d us -c "SELECT * FROM ${file}_state_metadata" | grep -v ".*M|" | sed -e 's/|.*//g' | paste -sd,);
  psql -Aqt -d us -c "SELECT jsonb_agg(row_to_json(fields)) FROM (SELECT name, ${columns} FROM ${file}_state2022 WHERE name NOT IN ('Puerto Rico')) fields;" > ~/american-geography/json/state/${file}_state.json
done

# counties (individual files)
files=('dp02' 'dp03' 'dp04' 'dp05')
for file in ${files[*]}; do
  columns=$(psql -Aqt -d us -c "SELECT * FROM ${file}_county_metadata" | grep -v ".*M|" | sed -e 's/|.*//g' | paste -sd,)
  psql -Aqt -d us -c "COPY (SELECT geo_id, name FROM ${file}_county2022) TO STDOUT DELIMITER E'\t'" | while IFS=$'\t' read -a array; do
    psql -Aqt -d us -c 'SELECT jsonb_agg(row_to_json(fields)) FROM (SELECT name, '"${columns}"' FROM '"${file}"'_county2022 WHERE geo_id = '\'${array[0]}\'') fields;' > ~/american-geography/json/county/${file}_${array[0]}.json
  done
done

# counties (top 3)
files=('dp02' 'dp03' 'dp04' 'dp05')
for file in ${files[*]}; do
  psql -Aqt -d us -c "SELECT column_name FROM information_schema.columns WHERE table_name = '${file}_county2022';" | grep -v "name" | grep -v "geo_id" | grep -v ".*m$" | while read column; do
    psql -Aqt -d us -c "SELECT jsonb_agg(row_to_json(fields)) FROM (SELECT name, ${column} FROM ${file}_county2022 WHERE ${column} NOT IN ('null','(X)') ORDER BY ${column}::REAL DESC LIMIT 3) fields;"
  done
done
```

Export to svg with json data for web.  
```shell
# states x counties
width=1920
height=1080
psql -d us -c "COPY (SELECT ST_XMin(geom), (-1 * ST_YMax(geom)), (ST_XMax(geom) - ST_XMin(geom)), (ST_YMax(geom) - ST_YMin(geom)), SUBSTRING(code_local,3,5), name FROM ne_10m_admin_1_states_provinces_lakes WHERE substring(code_local,1,2) = 'US') TO STDOUT DELIMITER E'\t'" | while IFS=$'\t' read -a array; do
  stateUpper=${array[5]// /}; state=${stateUpper,,};
  echo '<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" height="'${height}'" width="'${width}'" viewBox="'${array[0]}' '${array[1]}' '${array[2]}' '${array[3]}'">' > ~/svgeo/svg/states/${state}_counties.svg
  psql -d us -c "COPY (SELECT a.geoid, ST_AsSVG(b.geom, 1), a.name FROM county2022 a, ne_10m_admin_2_counties_lakes b WHERE a.geoid = b.code_local AND SUBSTRING(a.geoid,1,2) = '${array[4]}') TO STDOUT DELIMITER E'\t'" | while IFS=$'\t' read -a array; do
  columns=$(psql -Aqt -d us -c '\d county2022' | grep "zscore_" | sed -e 's/zscore_//g' -e 's/|.*//g' | paste -sd,)  
  zcolumns=$(psql -Aqt -d us -c "SELECT to_jsonb(inputs) FROM (SELECT $(echo ${columns} | tr ',' '\n' | sed -e 's/zscore_//g' -e "s/.*/CONCAT\('{county:', a\.\0, '|state:', b\.\0\, '|us:', c\.\0\, '}') AS \0/g" | paste -sd,) FROM county2022 a, state2022 b, us2022 c WHERE a.geoid = '${array[0]}' AND SUBSTRING(a.geoid,1,2) = b.geoid) inputs;")
    if [ -z "$zcolumns" ]; then
      echo '<path id="'${array[0]}'" d="'${array[1]}'" vector-effect="non-scaling-stroke" fill="#000" fill-opacity="0" stroke="#000" stroke-width="0.3px" stroke-linejoin="round" stroke-linecap="round"><title>'"${array[2]}"'</title></path>' >> ~/svgeo/svg/states/${state}_counties.svg
    else
      echo '<path id="'${array[0]}'" d="'${array[1]}'" vector-effect="non-scaling-stroke" fill="#000" fill-opacity="0" stroke="#000" stroke-width="0.3px" stroke-linejoin="round" stroke-linecap="round" data-json='\'${zcolumns}\''><title>'"${array[2]}"'</title></path>' >> ~/svgeo/svg/states/${state}_counties.svg
    fi
  done
  echo '</svg>' >> ~/svgeo/svg/states/${state}_counties.svg
done

# states x counties with zscores over 1.65
width=1920
height=1080
psql -d us -c "COPY (SELECT ST_XMin(geom), (-1 * ST_YMax(geom)), (ST_XMax(geom) - ST_XMin(geom)), (ST_YMax(geom) - ST_YMin(geom)), SUBSTRING(code_local,3,5), name FROM ne_10m_admin_1_states_provinces_lakes WHERE substring(code_local,1,2) = 'US') TO STDOUT DELIMITER E'\t'" | while IFS=$'\t' read -a array; do
  stateUpper=${array[5]// /}; state=${stateUpper,,};
  echo '<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" height="'${height}'" width="'${width}'" viewBox="'${array[0]}' '${array[1]}' '${array[2]}' '${array[3]}'">' > ~/svgeo/svg/states/${state}_counties.svg
  psql -d us -c "COPY (SELECT a.geoid, ST_AsSVG(b.geom, 1), a.name, a.pop FROM county2022 a, ne_10m_admin_2_counties_lakes b WHERE a.geoid = b.code_local AND SUBSTRING(a.geoid,1,2) = '${array[4]}') TO STDOUT DELIMITER E'\t'" | while IFS=$'\t' read -a array; do
    columns=$(psql -Aqt -d us -c "WITH b AS (SELECT $(psql -Aqt -d us -c '\d county2022' | grep "zscore_" | sed -e 's/|.*//g' | paste -sd,) FROM county2022 WHERE geoid = '${array[0]}') SELECT (x).key FROM (SELECT EACH(hstore(b)) x FROM b) q WHERE CAST((x).value AS VARCHAR) ~ '^[0-9\\\.]+$' AND ABS(CAST((x).value AS REAL)) >= 1.65;" | paste -sd,)
    zcolumns=$(psql -Aqt -d us -c "SELECT to_jsonb(inputs) FROM (SELECT $(echo ${columns} | tr ',' '\n' | sed -e 's/zscore_//g' -e "s/.*/CONCAT\('{county:', a\.\0, '|state:', b\.\0\, '|us:', c\.\0\, '}') AS \0/g" | paste -sd,) FROM county2022 a, state2022 b, us2022 c WHERE a.geoid = '${array[0]}' AND SUBSTRING(a.geoid,1,2) = b.geoid) inputs;")
    if [ -z "$zcolumns" ]; then
      echo '<path id="'${array[0]}'" d="'${array[1]}'" vector-effect="non-scaling-stroke" fill="#000" fill-opacity="0" stroke="#000" stroke-width="0.3px" stroke-linejoin="round" stroke-linecap="round"><title>'"${array[2]}"'</title></path>' >> ~/svgeo/svg/states/${state}_counties.svg
    else
      echo '<path id="'${array[0]}'" d="'${array[1]}'" vector-effect="non-scaling-stroke" fill="#000" fill-opacity="0" stroke="#000" stroke-width="0.3px" stroke-linejoin="round" stroke-linecap="round" data-json='\'${zcolumns}\''><title>'"${array[2]}"'</title></path>' >> ~/svgeo/svg/states/${state}_counties.svg
    fi
  done
  echo '</svg>' >> ~/svgeo/svg/states/${state}_counties.svg
done
```

Export geojson files for qgis.
```shell
# states
psql -qAtX -d us -c '\d state2022;' | grep -v "shape" | grep -v "geoid" | grep -v "name" | grep -v "zscore_" | sed -e 's/|.*//g' | while read column; do
  psql -d us -c "COPY (SELECT jsonb_build_object('type', 'FeatureCollection', 'features', jsonb_agg(feature)) FROM (SELECT jsonb_build_object('type', 'Feature', 'id', geoid, 'geometry', ST_AsGeoJSON(shape)::jsonb, 'properties', to_jsonb(inputs) - 'shape' - 'geoid') AS feature FROM (SELECT shape, geoid, name, RANK() OVER (ORDER BY ${column}::real DESC) rank, ${column}::real FROM state2022 WHERE ${column}::text ~ '^[0-9\\\.]+$') inputs) features) TO STDOUT;" > geojson/state/state2022_${column}.geojson
done
```

Export county quartiles (percentage columns only).  
```shell
# which columns are valid?
files=('dp02' 'dp03' 'dp04' 'dp05')
for file in ${files[*]}; do
  psql -qAtX -d us -c "\d ${table}"| sed -e 's/|.*//g' | grep 'pe' | grep -v 'zscore' | while read column; do
    table=${file}_county2022
    psql -qAtX -d us -c "SELECT COUNT(${column}) FROM ${table} WHERE ${column} IS NOT NULL"
  done
done

files=('dp02' 'dp03' 'dp04' 'dp05')
for file in ${files[*]}; do
  table=${file}_county2022
  psql -qAtX -d us -c "\d ${table}"| sed -e 's/|.*//g' | grep 'pe' | grep -v 'zscore' | while read column; do
    psql -d us -c "COPY (SELECT jsonb_build_object('type', 'FeatureCollection', 'features', jsonb_agg(feature)) FROM (SELECT jsonb_build_object('type', 'Feature', 'id', quartile, 'geometry', ST_AsGeoJSON(ST_Transform(geom,4326))::jsonb, 'properties', to_jsonb(inputs) - 'geom') AS feature FROM (WITH stats AS (SELECT DISTINCT PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY ${column}::real) q1, PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY ${column}::real) q2, PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY ${column}::real) q3 FROM ${table} WHERE ${column}::text ~ '^[0-9\\\.]+$' AND ${column}::real <= 100) SELECT 'q1' AS quartile, MIN(${column}::real) AS min, MAX(${column}::real) AS max, ST_Union(ST_Buffer(b.geom,0)) AS geom FROM ${table} a, ne_10m_admin_2_counties_lakes b, stats WHERE a.${column}::real < stats.q1 AND SUBSTRING(a.geo_id,10,5) = b.code_local UNION ALL SELECT 'q2' AS quartile, MIN(${column}::real) AS min, MAX(${column}::real) AS max, ST_Union(ST_Buffer(b.geom,0)) AS geom FROM ${table} a, ne_10m_admin_2_counties_lakes b, stats WHERE a.${column}::real >= stats.q1 AND a.${column}::real < stats.q2 AND SUBSTRING(a.geo_id,10,5) = b.code_local UNION ALL SELECT 'q3' AS quartile, MIN(${column}::real) AS min, MAX(${column}::real) AS max, ST_Union(ST_Buffer(b.geom,0)) AS geom FROM ${table} a, ne_10m_admin_2_counties_lakes b, stats WHERE a.${column}::real >= stats.q2 AND a.${column}::real < stats.q3 AND SUBSTRING(a.geo_id,10,5) = b.code_local UNION ALL SELECT 'q4' AS quartile, MIN(${column}::real) AS min, MAX(${column}::real) AS max, ST_Union(ST_Buffer(b.geom,0)) AS geom FROM ${table} a, ne_10m_admin_2_counties_lakes b, stats WHERE a.${column}::real >= stats.q3 AND SUBSTRING(a.geo_id,10,5) = b.code_local) inputs) features) TO STDOUT;" > county_quartile_${column}.geojson
  done
done
```

Export county kmean clusters (percentage columns only).  
```shell
files=('dp02' 'dp03' 'dp04' 'dp05')
for file in ${files[*]}; do
  table=${file}_county2022
  psql -qAtX -d us -c "\d ${table}" | sed -e 's/|.*//g' | grep 'pe' | grep -v 'zscore' | while read column; do
    psql -d us -c "COPY (SELECT jsonb_build_object('type', 'FeatureCollection', 'features', jsonb_agg(feature)) FROM (SELECT jsonb_build_object('type', 'Feature', 'id', cluster_id, 'geometry', ST_AsGeoJSON(ST_Transform(geom,4326))::jsonb, 'properties', to_jsonb(inputs) - 'geom') AS feature FROM (WITH stats AS (SELECT ST_ClusterKMeans(ST_Force4D(ST_Transform(ST_Force3D(geom), 4978), mvalue:=a.${column}::real), 4) OVER () AS cluster_id, a.geo_id, a.${column}, ST_Union(ST_Buffer(b.geom,0)) geom FROM ${table} a, ne_10m_admin_2_counties_lakes b WHERE SUBSTRING(a.geo_id,10,5) = b.code_local AND a.${column}::text ~ '^[0-9\\\.]+$' AND a.${column}::real <= 100) SELECT a.cluster_id, (SELECT MIN(${column}) FROM stats) AS min, (SELECT MAX(${column}) FROM stats) AS max, geom FROM stats) inputs) features) TO STDOUT;" > county_cluster_${column}.geojson
  done
done
```

Export county equal intervals (percentage columns only).  
```shell
# update cluster_id
files=('dp02' 'dp03' 'dp04' 'dp05')
for file in ${files[*]}; do
  table=${file}_county2022
  psql -qAtX -d us -c "\d ${table}" | sed -e 's/|.*//g' | grep 'pe' | grep -v 'zscore' | while read column; do
    psql -d us -c "ALTER TABLE ${table} DROP COLUMN IF EXISTS cluster_id; ALTER TABLE ${table} ADD COLUMN cluster_id real;"
    psql -d us -c "WITH b AS (SELECT MIN(${column}::real) AS min, MAX(${column}::real) AS max FROM ${table} WHERE ${column}::text ~ '^[0-9\\\.]+$' AND ${column}::real <= 100) UPDATE ${table} a SET cluster_id = width_bucket(a.${column}::real, b.min, b.max, 4) FROM b WHERE ${column}::text ~ '^[0-9\\\.]+$' AND ${column}::real <= 100;"
  done
done

# export
files=('dp02' 'dp03' 'dp04' 'dp05')
for file in ${files[*]}; do
  table=${file}_county2022
  psql -qAtX -d us -c "\d ${table}" | sed -e 's/|.*//g' | grep 'pe' | grep -v 'zscore' | while read column; do
    psql -d us -c "COPY (SELECT jsonb_build_object('type', 'FeatureCollection', 'features', jsonb_agg(feature)) FROM (SELECT jsonb_build_object('type', 'Feature', 'id', cluster_id, 'geometry', ST_AsGeoJSON(ST_Transform(geom,4326))::jsonb, 'properties', to_jsonb(inputs) - 'geom') AS feature FROM (WITH stats AS (SELECT a.cluster_id, MIN(a.${column}::real) AS MIN, MAX(a.${column}::real) AS MAX, ST_Union(ST_Buffer(b.geom,0)) AS geom FROM ${table} a, ne_10m_admin_2_counties_lakes b WHERE SUBSTRING(a.geo_id,10,5) = b.code_local AND a.${column}::text ~ '^[0-9\\\.]+$' AND a.${column}::real <= 100 GROUP BY a.cluster_id) SELECT cluster_id, min, max, geom FROM stats) inputs) features) TO STDOUT;" > county_cluster_${column}.geojson
  done
done
```

Export top 10 
```shell
# state top10
psql -qAtX -d us -c '\d state2022;' | grep -v "shape" | grep -v "geoid" | grep -v "name" | grep -v "zscore_" | sed -e 's/|.*//g' | while read column; do
  psql -d us -c "COPY (SELECT jsonb_build_object('type', 'FeatureCollection', 'features', jsonb_agg(feature)) FROM (SELECT jsonb_build_object('type', 'Feature', 'id', geoid, 'geometry', ST_AsGeoJSON(ST_Transform(geom,4326))::jsonb, 'properties', to_jsonb(inputs) - 'geom' - 'geoid') AS feature FROM (SELECT b.geom, a.geoid, a.name, RANK() OVER (ORDER BY a.${column}::real DESC) rank, a.${column} AS column, '${column}' AS column_name FROM state2022 a, ne_10m_admin_1_states_provinces_lakes b WHERE a.${column}::text ~ '^[0-9\\\.]+$' AND CONCAT('US',a.geoid) = b.code_local ORDER BY a.${column}::real DESC LIMIT 10) inputs) features) TO STDOUT;" > geojson/state/state_top10_${column}.geojson
done

# county top10
psql -qAtX -d us -c '\d county2022;' | grep -v "shape" | grep -v "geoid" | grep -v "name" | grep -v "zscore_" | sed -e 's/|.*//g' | while read column; do
  psql -d us -c "COPY (SELECT jsonb_build_object('type', 'FeatureCollection', 'features', jsonb_agg(feature)) FROM (SELECT jsonb_build_object('type', 'Feature', 'id', geoid, 'geometry', ST_AsGeoJSON(ST_Transform(geom,4326))::jsonb, 'properties', to_jsonb(inputs) - 'geom' - 'geoid') AS feature FROM (SELECT b.geom, a.geoid, a.name, b.region AS state, RANK() OVER (ORDER BY a.${column}::real DESC) rank, a.${column} AS column, '${column}' AS column_name FROM county2022 a, ne_10m_admin_2_counties_lakes b WHERE a.${column}::text ~ '^[0-9\\\.]+$' AND a.geoid = b.code_local ORDER BY a.${column}::real DESC LIMIT 10) inputs) features) TO STDOUT;" > geojson/county/county_top10_${column}.geojson
done
```

## Misc

How to select your own variables and create tables.  
```shell
columns='DP05_0001E AS pop, DP05_0005PE AS age_5_under, DP05_0006PE AS age_5_9, DP05_0007PE AS age_10_14, DP05_0008PE AS age_15_19, DP05_0009PE AS age_20_24, DP05_0010PE AS age_25_34, DP05_0011PE AS age_35_44, DP05_0012PE AS age_45_54, DP05_0013PE AS age_55_59, DP05_0014PE AS age_60_64, DP05_0015PE AS age_65_74, DP05_0016PE AS age_75_84, DP05_0017PE AS age_85_over, DP05_0018E AS age_median, DP05_0019PE AS age_under18, DP05_0020PE AS age_16_over, DP05_0021PE AS age_18_over, DP05_0022PE AS age_21_over, DP05_0023PE AS age_62_over, DP05_0024PE AS age_65_over, DP05_0035PE AS race_mixed, DP05_0037PE AS race_white, DP05_0038PE AS race_black, DP05_0039PE AS race_native, DP05_0044PE AS race_asian, DP05_0052PE AS race_pacific, DP05_0071PE AS race_hispanic, DP05_0086E AS housing_total, DP03_0004PE AS employed, DP03_0005PE AS unemployed, DP03_0027PE AS job_bus_sci_art, DP03_0028PE AS job_service, DP03_0029PE AS job_sales, DP03_0030PE AS job_construction, DP03_0031PE AS job_prod_transpo, DP03_0033PE AS in_agri_mining, DP03_0034PE AS in_construction, DP03_0035PE AS in_manufacturing, DP03_0036PE AS in_wholesale, DP03_0037PE AS in_retail, DP03_0038PE AS in_transportation, DP03_0039PE AS in_information, DP03_0040PE AS in_finance, DP03_0041PE AS in_professional, DP03_0042PE AS in_education_health, DP03_0043PE AS in_arts_food, DP03_0044PE AS in_other, DP03_0045PE AS in_public_admin, DP03_0047PE AS class_private, DP03_0048PE AS class_govt, DP03_0049PE AS class_selfemployed, DP03_0050PE AS class_unpaid, DP03_0052PE AS income_10000_less, DP03_0053PE AS income_10000_14999, DP03_0054PE AS income_15000_24999, DP03_0055PE AS income_25000_34999, DP03_0056PE AS income_35000_49999, DP03_0057PE AS income_50000_74999, DP03_0058PE AS income_75000_99999, DP03_0059PE AS income_100000_149999, DP03_0060PE AS income_150000_199999, DP03_0061PE AS income_200000_more, DP03_0062E AS income_median, DP03_0063E AS income_mean, DP03_0088E AS income_percapita, DP03_0065E AS earnings_mean, DP03_0092E AS earnings_median, DP03_0093E AS earnings_male, DP03_0094E AS earnings_female, DP04_0001E AS housing_units, DP04_0002PE AS housing_occupied, DP04_0003PE AS housing_vacant, DP04_0004PE AS vacancy_homeowner, DP04_0005PE AS vacancy_rental, DP04_0007PE AS units_1_detached, DP04_0008PE AS units_1_attached, DP04_0009PE AS units_2, DP04_0010PE AS units_3_4, DP04_0011PE AS units_5_9, DP04_0012PE AS units_10_19, DP04_0013PE AS units_20_more, DP04_0014PE AS units_mobile, DP04_0015PE AS units_boat_rv_van, DP04_0017PE AS built_2014_later, DP04_0018PE AS built_2010_2013, DP04_0019PE AS built_2000_2009, DP04_0020PE AS built_1990_1999, DP04_0021PE AS built_1980_1989, DP04_0022PE AS built_1970_1979, DP04_0023PE AS built_1960_1969, DP04_0024PE AS built_1950_1959, DP04_0025PE AS built_1940_1949, DP04_0026PE AS built_1939_earlier, DP04_0028PE AS rooms_1, DP04_0029PE AS rooms_2, DP04_0030PE AS rooms_3, DP04_0031PE AS rooms_4, DP04_0032PE AS rooms_5, DP04_0033PE AS rooms_6, DP04_0034PE AS rooms_7, DP04_0035PE AS rooms_8, DP04_0036PE AS rooms_9, DP04_0037E AS rooms_median, DP04_0046PE AS occupied_owner, DP04_0047PE AS occupied_renter, DP04_0081PE AS value_50000_less, DP04_0082PE AS value_50000_99999, DP04_0083PE AS value_100000_149000, DP04_0084PE AS value_150000_199999, DP04_0085PE AS value_200000_299999, DP04_0086PE AS value_300000_499999, DP04_0087PE AS value_500000_999999, DP04_0088PE AS value_1000000_more, DP04_0089E AS value_median, DP04_0127PE AS rent_500_less, DP04_0128PE AS rent_500_999, DP04_0129PE AS rent_1000_1499, DP04_0130PE AS rent_1500_1999, DP04_0131PE AS rent_2000_2499, DP04_0132PE AS rent_2500_2999, DP04_0133PE AS rent_3000_more, DP04_0134E AS rent_median, DP04_0137PE AS grapi_15_less, DP04_0138PE AS grapi_15_19, DP04_0139PE AS grapi_20_24, DP04_0140PE AS grapi_25_29, DP04_0141PE AS grapi_30_34, DP04_0142PE AS grapi_35_more, DP02_0060PE AS edu_9_less, DP02_0061PE AS edu_9_12, DP02_0062PE AS edu_highschool, DP02_0063PE AS edu_college_nodegree, DP02_0064PE AS edu_associate_degree, DP02_0065PE AS edu_bachelor_degree, DP02_0066PE AS edu_grad_degree, DP02_0067PE AS edu_highschool_higher, DP02_0068PE AS edu_bachelor_higher, DP02_0125PE AS ancestry_american, DP02_0126PE AS ancestry_arab, DP02_0127PE AS ancestry_czech, DP02_0128PE AS ancestry_danish, DP02_0129PE AS ancestry_dutch, DP02_0130PE AS ancestry_english, DP02_0131PE AS ancestry_french, DP02_0132PE AS ancestry_frenchcanadian, DP02_0133PE AS ancestry_german, DP02_0134PE AS ancestry_greek, DP02_0135PE AS ancestry_hungarian, DP02_0136PE AS ancestry_irish, DP02_0137PE AS ancestry_italian, DP02_0138PE AS ancestry_lithuanian, DP02_0139PE AS ancestry_norwegian, DP02_0140PE AS ancestry_polish, DP02_0141PE AS ancestry_portuguese, DP02_0142PE AS ancestry_russian, DP02_0143PE AS ancestry_scottish_irish, DP02_0144PE AS ancestry_scottish, DP02_0145PE AS ancestry_slovak, DP02_0146PE AS ancestry_subsaharan_african, DP02_0147PE AS ancestry_swedish, DP02_0148PE AS ancestry_swiss, DP02_0149PE AS ancestry_ukranian, DP02_0150PE AS ancestry_welsh, DP02_0151PE AS ancestry_west_indian'
# us
psql -d us -c "DROP TABLE IF EXISTS us2022; CREATE TABLE us2022 AS SELECT ${columns} FROM dp02_us2022, dp03_us2022, dp04_us2022, dp05_us2022;"
# state
psql -d us -c "DROP TABLE IF EXISTS state2022; CREATE TABLE state2022 AS SELECT a.shape, a.geoid, a.name, ${columns} FROM state a, dp02_state2022 b, dp03_state2022 c, dp04_state2022 d, dp05_state2022 e WHERE a.geoid = SUBSTRING(b.geo_id,10,2) AND a.geoid = SUBSTRING(c.geo_id,10,2) AND a.geoid = SUBSTRING(d.geo_id,10,2) AND a.geoid = SUBSTRING(e.geo_id,10,2);"
# county
psql -d us -c "DROP TABLE IF EXISTS county2022; CREATE TABLE county2022 AS SELECT a.shape, a.geoid, a.namelsad AS name, ${columns} FROM county a, dp02_county2022 b, dp03_county2022 c, dp04_county2022 d, dp05_county2022 e WHERE a.geoid = SUBSTRING(b.geo_id,10,5) AND a.geoid = SUBSTRING(c.geo_id,10,5) AND a.geoid = SUBSTRING(d.geo_id,10,5) AND a.geoid = SUBSTRING(e.geo_id,10,5);"
# place
psql -d us -c "DROP TABLE IF EXISTS place2022; CREATE TABLE place2022 AS SELECT a.shape, a.geoid, a.namelsad AS name, ${columns} FROM place a, dp02_place2022 b, dp03_place2022 c, dp04_place2022 d, dp05_place2022 e WHERE a.geoid = SUBSTRING(b.geo_id,10,7) AND a.geoid = SUBSTRING(c.geo_id,10,7) AND a.geoid = SUBSTRING(d.geo_id,10,7) AND a.geoid = SUBSTRING(e.geo_id,10,7);"
# tract
psql -d us -c "DROP TABLE IF EXISTS tract2022; CREATE TABLE tract2022 AS SELECT a.shape, a.geoid, a.namelsad AS name, ${columns} FROM census_tract a, dp02_tract2022 b, dp03_tract2022 c, dp04_tract2022 d, dp05_tract2022 e WHERE a.geoid = SUBSTRING(b.geo_id,10,11) AND a.geoid = SUBSTRING(c.geo_id,10,11) AND a.geoid = SUBSTRING(d.geo_id,10,11) AND a.geoid = SUBSTRING(e.geo_id,10,11);"
# puma
psql -d us -c "DROP TABLE IF EXISTS puma2022; CREATE TABLE puma2022 AS SELECT DISTINCT ON (b.name) a.geom AS shape, a.geoid20 AS geoid, a.namelsad20 AS name, ${columns} FROM puma a, dp02_puma2022 b, dp03_puma2022 c, dp04_puma2022 d, dp05_puma2022 e WHERE a.geoid20 = SUBSTRING(b.geo_id,10,7) AND a.geoid20 = SUBSTRING(c.geo_id,10,7) AND a.geoid20 = SUBSTRING(d.geo_id,10,7) AND a.geoid20 = SUBSTRING(e.geo_id,10,7);"

# index
geographies=('state' 'county' 'place' 'tract' 'puma')
for geography in ${geographies[*]}; do
  psql -d us -c "ALTER TABLE ${geography}2022 ADD PRIMARY KEY (geoid); CREATE INDEX ${geography}2022_gid ON ${geography}2022 USING GIST (shape);"
done
```

### OpenStreetMap

See [postgis-cookbook](https://github.com/geographyclub/postgis-cookbook) for more.

## References

Release schedule: https://www.census.gov/programs-surveys/acs/news/data-releases/2022/release-schedule.html

Technical documentation: https://www.census.gov/programs-surveys/acs/technical-documentation.html

Table shells: https://www.census.gov/programs-surveys/acs/technical-documentation/table-shells.html

Areas published: https://www.census.gov/programs-surveys/acs/geography-acs/areas-published.html

Understanding geoids: https://www.census.gov/programs-surveys/geography/guidance/geo-identifiers.html

