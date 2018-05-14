# Transformació oficial de datum ICGC amb PostGIS

Mètodes de transformació disponibles:

* Transformació afí directa EPSG:23031 => EPSG:25831
* Usar fitxer de malla NTv2 a PROJ.4

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

## Transformació afí

### A nivell de PROJ.4

No està suportada:

http://osgeo-org.1560.x6.nabble.com/How-to-apply-an-affine-transformation-td3842074.html

https://github.com/OSGeo/proj.4/issues/535


### A nivell de SQL PostGIS

Paràmetres oficials de la transformació d'ED50 a ETRS89:

* Tx: -129.549 m
* Ty: -208.185 m
* μ: 0.0000015504
* α: -1,56504 " = -0.000007587528034836682 rad

Funció de transformació Afí segons el manual de PostGIS:

```sql
ST_Affine(geom, a, b, d, e, xoff, yoff)
```

Substituïnt valors:

* a = (1+μ)·cos(α) = 1.0000015503712145
* b = -(1+μ)·sin(α) = 0.000007587539798467343
* d = (1+μ)·sin(α) = -0.000007587539798467343
* e = (1+μ)·cos(α) = 1.0000015503712145
* xoff = Tx = -129.549
* yoff = Ty = -208.185

ST_Affine:

```sql
SELECT id, ST_AsEWKT(geom), ST_AsEWKT(ST_Affine(geom, 1.0000015503712145, 0.000007587539798467343, -0.000007587539798467343, 1.0000015503712145, -129.549, -208.185)) FROM ed50_test_points;
```

Anàlogament, es pot definir la transformació ETRS89 => ED50 partint dels paràmetres donats per l'ICGC:

* Tx: 129.547 m
* Ty: 208.186 m
* μ: -0.0000015504
* α: 1,56504 " = 0.000007587528034836682 rad

Als paràmetres de la matriu de transformació afí:

* a = (1+μ)·cos(α) = 0.9999984495712146
* b = -(1+μ)·sin(α) = -0.000007587516271060413
* d = (1+μ)·sin(α) = 0.000007587516271060413
* e = (1+μ)·cos(α) = 0.9999984495712146
* xoff = Tx = 129.547
* yoff = Ty = 208.186

ST_Affine:

```sql
SELECT id, ST_AsEWKT(geom), ST_AsEWKT(ST_Affine(geom, 0.9999984495712146, -0.000007587516271060413, 0.000007587516271060413, 0.9999984495712146, 129.547, 208.186)) FROM etrs89_test_points;
```

https://gis.stackexchange.com/questions/34612/changing-srid-of-existing-data-in-postgis

https://postgis.net/docs/UpdateGeometrySRID.html

https://postgis.net/docs/ST_SetSRID.html


## Malla NTv2

https://github.com/geomatico/postgis2.0-advanced/blob/master/tema2.rst

```sql
SELECT * FROM spatial_ref_sys WHERE srid=23031 OR srid=25831;
```



Recursos:
* https://www.avantgeo.com/transformar-de-ed50-a-etrs89-en-postgis/
* http://www.icgc.cat/ca/Administracio-i-empresa/Eines/Transformacio-de-coordenades-i-formats/ETRS89/Accions/Forum-ETRS892/Problemes-transformacio-ED50-a-ETRS89
