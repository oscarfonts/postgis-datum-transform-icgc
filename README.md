# Transformació oficial de datum ICGC amb PostGIS

Mètodes de transformació disponibles:

1. Transformació oficial directa entre EPSG:23031 i EPSG:25831
2. Usant fitxer de malla NTv2

http://www.icgc.cat/ca/Administracio-i-empresa/Eines/Transformacio-de-coordenades-i-formats

## Punts de test

Obtinguts del document "Transformació bidimensional de semblança entre ED50 i ETRS89":
http://www.icgc.cat/ca/content/download/48760/337896/version/1/file/H2D_v8.pdf

* Transformació directa EPSG:23031 => EPSG:25831:

      300000.000 4500000.000 => 299905.060 4499796.515
      315000.000 4740000.000 => 314906.904 4739796.774
      520000.000 4680000.000 => 519906.767 4679795.125
      420000.000 4600000.000 => 419906.005 4599795.760

* Transformació inversa EPSG:25831 => EPSG:23031:

      300000.000 4500000.000 => 300094.938 4500203.485
      315000.000 4740000.000 => 315093.094 4740203.227
      520000.000 4680000.000 => 520093.231 4680204.876
      420000.000 4600000.000 => 420093.993 4600204.241

Podem crear una BDD de test a PostGIS amb aquests punts de test:

Crear BDD amb superusuari `postgres`:

```sql
CREATE USER datumtest LOGIN PASSWORD 'datumtest' NOSUPERUSER INHERIT NOCREATEDB NOCREATEROLE;
CREATE DATABASE datumtest OWNER datumtest;
\c datumtest
CREATE EXTENSION postgis;
```

Després, connectant-se amb l'usari `datumtest`:

```sql
CREATE TABLE ed50_test_points (id serial, geom geometry(Point, 23031));
INSERT INTO ed50_test_points (geom) VALUES
    (ST_GeomFromText('POINT(300000.000 4500000.000)', 23031)),
    (ST_GeomFromText('POINT(315000.000 4740000.000)', 23031)),
    (ST_GeomFromText('POINT(520000.000 4680000.000)', 23031)),
    (ST_GeomFromText('POINT(420000.000 4600000.000)', 23031));

CREATE TABLE etrs89_test_points (id serial, geom geometry(Point, 25831));
INSERT INTO etrs89_test_points (geom) VALUES
    (ST_GeomFromText('POINT(300000.000 4500000.000)', 25831)),
    (ST_GeomFromText('POINT(315000.000 4740000.000)', 25831)),
    (ST_GeomFromText('POINT(520000.000 4680000.000)', 25831)),
    (ST_GeomFromText('POINT(420000.000 4600000.000)', 25831));
```

## Transformació oficial directa entre EPSG:23031 i EPSG:25831

### A nivell de PROJ.4

La millor opció seria incloure la transformació com una definició de PROJ.4 a la taula `spatial_ref_sys`. Malauradament, PROJ.4 no suporta transformacions d'aquest tipus entre sistemes projectats:

http://osgeo-org.1560.x6.nabble.com/How-to-apply-an-affine-transformation-td3842074.html

https://github.com/OSGeo/proj.4/issues/535

### A nivell de SQL

En canvi, PostGIS sí que proporciona una funció de tansformació afí, `ST_Affine`, que es pot aplicar a qualsevol geometria:

```sql
ST_Affine(geom, a, b, d, e, xoff, yoff)
```

Veure: https://postgis.net/docs/ST_Affine.html

Consultant el document oficial de la transformació de semblança de l'ICGC n'obtenim els següents paràmetres de transformació entre ED50 i ETRS89 (per a la projecció UTM, fus 31 N):

* Tx: -129.549 m
* Ty: -208.185 m
* μ: 0.0000015504
* α: -1,56504 " = -0.000007587528034836682 rad

Substituïnt valors als paràmetres tal com els anomena PostGIS, obtindriem:

* a = (1+μ)·cos(α) = 1.0000015503712145
* b = -(1+μ)·sin(α) = 0.000007587539798467343
* d = (1+μ)·sin(α) = -0.000007587539798467343
* e = (1+μ)·cos(α) = 1.0000015503712145
* xoff = Tx = -129.549
* yoff = Ty = -208.185

Obtenim la funció ST_Affine:

```sql
ST_Affine(geom, 1.0000015503712145, 0.000007587539798467343, -0.000007587539798467343, 1.0000015503712145, -129.549, -208.185)
```

Podem escriure una funció PLPGSQL que substitueixi totes les geometries d'una taula en EPSG:23031 cap a EPSG:25831:

```sql
-- Function: public.icgc_23031_to_25831(character varying, character varying, character varying)

-- DROP FUNCTION public.icgc_23031_to_25831(character varying, character varying, character varying);

CREATE OR REPLACE FUNCTION public.icgc_23031_to_25831(
    schema_name character varying,
    table_name character varying,
    column_name character varying)
  RETURNS text AS
$BODY$
DECLARE
	real_schema name;
	src_srid integer;
	dst_srid integer;
BEGIN
	-- Set SRID values
	src_srid := 23031;
	dst_srid := 25831;
	
	-- Find, check or fix schema_name
	IF ( schema_name != '' ) THEN
		real_schema = schema_name;
	ELSE
		SELECT INTO real_schema current_schema()::text;
	END IF;

	-- Check original SRID
	IF (SELECT count(*) = 0 FROM geometry_columns WHERE f_table_schema = real_schema AND f_table_name = table_name AND f_geometry_column = column_name AND srid = src_srid) THEN
		RAISE EXCEPTION 'table original SRID not matching required SRID %', src_srid;
		RETURN false;
	END IF;

	-- Set new SRID to table metadata
	PERFORM updategeometrysrid(schema_name, table_name, column_name, dst_srid);

	-- Reproject using affine transform
	EXECUTE 'UPDATE ' || quote_ident(real_schema) || '.' || quote_ident(table_name) || ' SET ' || quote_ident(column_name) || ' = ST_Affine('|| quote_ident(column_name) ||', 1.0000015503712145, 0.000007587539798467343, -0.000007587539798467343, 1.0000015503712145, -129.549, -208.185)';

	RETURN real_schema || '.' || table_name || '.' || column_name ||' datum transformed to ' || dst_srid::text;
END;
$BODY$
  LANGUAGE plpgsql VOLATILE STRICT
  COST 100;
ALTER FUNCTION public.icgc_23031_to_25831(character varying, character varying, character varying)
  OWNER TO datumtest;
COMMENT ON FUNCTION public.icgc_23031_to_25831(character varying, character varying, character varying) IS 'args: schema_name, table_name, column_name - Transforms geometries in a table from EPSG:23031 to EPSG:25831 using the official catalan affine transform';
```

Aquesta funció s'usa així:

```sql
SELECT icgc_23031_to_25831('public', 'table_name', 'geom');
```


Anàlogament, es pot definir la transformació inversa, ETRS89 => ED50, partint dels paràmetres donats per l'ICGC:

* Tx: 129.547 m
* Ty: 208.186 m
* μ: -0.0000015504
* α: 1,56504 " = 0.000007587528034836682 rad

Els paràmetres de la matriu de transformació afí serien:

* a = (1+μ)·cos(α) = 0.9999984495712146
* b = -(1+μ)·sin(α) = -0.000007587516271060413
* d = (1+μ)·sin(α) = 0.000007587516271060413
* e = (1+μ)·cos(α) = 0.9999984495712146
* xoff = Tx = 129.547
* yoff = Ty = 208.186

La funció ST_Affine quedaria escrita com:

```sql
ST_Affine(geom, 0.9999984495712146, -0.000007587516271060413, 0.000007587516271060413, 0.9999984495712146, 129.547, 208.186)
```

I, de la mateixa manera, es podria escriure una funció PLSQL per transformar inversament.


## Usant fitxer de malla NTv2

En aquest cas, ens haurem de baixar el fitxer `100800401.gsb` de la web de l'ICGC (http://www.icgc.cat/ca/Administracio-i-empresa/Eines/Transformacio-de-coordenades-i-formats), i copiar-la al lloc on tinguem instal·lat PROJ.4.

A Ubuntu 16.04 la ubicació és `/usr/share/proj`:

```bash
sudo cp 100800401.gsb /usr/share/proj
```

Llavors cal modificar la definició dels SRS a la taula `spatial_ref_sys`:

```sql
update spatial_ref_sys set proj4text = '+proj=utm +zone=31 +ellps=intl +units=m +no_defs +nadgrids=100800401.gsb' where srid = 23031;
update spatial_ref_sys set proj4text = '+proj=longlat +ellps=intl +no_defs +nadgrids=100800401.gsb' where srid = 4230;
```

Comprovació fent servir les taules d'exemple:

```sql
SELECT ST_AsEWKT(geom), ST_AsEWKT(ST_Transform(geom, 25831)) from ed50_test_points;
SELECT ST_AsEWKT(geom), ST_AsEWKT(ST_Transform(geom, 23031)) from etrs89_test_points;
```
