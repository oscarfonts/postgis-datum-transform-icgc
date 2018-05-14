# Transformació oficial de datum ICGC amb PostGIS

Mètodes de transformació disponibles:

* Transformació afí directa EPSG:23031 => EPSG:25831
* Usar fitxer de malla NTv2 a PROJ.4

http://www.icgc.cat/ca/Administracio-i-empresa/Eines/Transformacio-de-coordenades-i-formats


## Punts de test

Obtinguts del document "Transformació bidimensional de semblança entre ED50 i ETRS89":
http://www.icgc.cat/ca/content/download/48760/337896/version/1/file/H2D_v8.pdf

* Transformació directa EPSG:23031 => EPSG:25831:

    POINT(300000.000 4500000.000) => POINT(299905.060 4499796.515)
    POINT(315000.000 4740000.000) => POINT(314906.904 4739796.774)
    POINT(520000.000 4680000.000) => POINT(519906.767 4679795.125)
    POINT(420000.000 4600000.000) => POINT(419906.005 4599795.760)

* Transformació inversa EPSG:25831 => EPSG:23031:

    POINT(300000.000 4500000.000) => POINT(300094.938 4500203.485)
    POINT(315000.000 4740000.000) => POINT(315093.094 4740203.227)
    POINT(520000.000 4680000.000) => POINT(520093.231 4680204.876)
    POINT(420000.000 4600000.000) => POINT(420093.993 4600204.241)

### Crear BDD amb els punts de test

Crear BDD:

```sql
CREATE USER datumtest LOGIN PASSWORD 'datumtest' NOSUPERUSER INHERIT NOCREATEDB NOCREATEROLE;
CREATE DATABASE datumtest OWNER datumtest;
\c datumtest
CREATE EXTENSION postgis;
```

Crear taules amb punts de test:

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

## Malla NTv2

https://github.com/geomatico/postgis2.0-advanced/blob/master/tema2.rst

```sql
SELECT * FROM spatial_ref_sys WHERE srid=23031 OR srid=25831;
```

Recursos:
* https://www.avantgeo.com/transformar-de-ed50-a-etrs89-en-postgis/
* http://www.icgc.cat/ca/Administracio-i-empresa/Eines/Transformacio-de-coordenades-i-formats/ETRS89/Accions/Forum-ETRS892/Problemes-transformacio-ED50-a-ETRS89


## Transformació afí

### A nivell de PROJ.4

http://osgeo-org.1560.x6.nabble.com/How-to-apply-an-affine-transformation-td3842074.html

https://github.com/OSGeo/proj.4/issues/535


### A nivell de SQL PostGIS

https://postgis.net/docs/ST_Affine.html

https://gis.stackexchange.com/questions/34612/changing-srid-of-existing-data-in-postgis

https://postgis.net/docs/UpdateGeometrySRID.html

https://postgis.net/docs/ST_SetSRID.html

