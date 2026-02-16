# postgresql_sgp4

`postgresql_sgp4` is a PostgreSQL extension that embeds an SGP4 orbital propagator directly inside the database engine, allowing satellite position computation from TLE data using pure SQL.

The extension itself is installed in PostgreSQL under the name `sgp4`.

```sql
CREATE EXTENSION sgp4;
```

## Overview

This project is based on a fork of the C implementation of SGP4 from:

https://github.com/aholinch/sgp4

The original code was adapted to operate as a PostgreSQL C extension. The following changes were made:

- Integration with PostgreSQL’s extension API (`fmgr`, `PG_MODULE_MAGIC`, etc.).
- Wrapping of the SGP4 and TLE parsing logic into SQL-callable functions.
- Conversion of PostgreSQL `timestamp` values into ISO-8601 strings for orbital propagation.
- Implementation of TEME → Earth-fixed coordinate conversion.
- Replacement of the legacy GMST computation with an Earth Rotation Angle (ERA) based implementation to improve rotational accuracy.
- Exposure of results as WKT (`POINT Z(lon lat alt)`) for easy integration with PostGIS.

The extension allows computing a satellite’s geographic position (WGS84 latitude, longitude, and altitude) directly within SQL queries.

## Disclaimer

This extension is intended for operational, visualization, and database-driven spatial analysis use cases. It is **not** intended for scientific, astrometric, or high-precision geodetic applications.

Although the Earth rotation model was improved by replacing the legacy GMST computation with an ERA-based approach, the implementation still omits several corrections used in modern high-precision libraries such as Skyfield, including:

- Full nutation and precession models (IAU 2000/2006)
- Equation of the equinoxes
- Polar motion
- Real-time UT1 corrections (ΔUT1)

As a result, small but measurable differences may exist when compared to high-precision astronomical toolkits. These differences can reach fractions of a degree in longitude depending on epoch and orbital configuration.

For scientific-grade orbit determination, precise geodesy, or reference-frame–critical applications, specialized astronomical libraries should be used instead.

Additionally, this extension has not been extensively or rigorously tested across all possible parameters values, orbital regimes, edge cases, and PostgreSQL deployment scenarios. It is provided as-is and should be used at your own risk. Users are responsible for validating results against trusted references before relying on them in production, safety-critical, scientific, or regulatory contexts.

## Build

```commandline
make
```

```commandline
make install
```

## PostGIS integration

To integrate with postgis, declare this function.

```sql
CREATE OR REPLACE FUNCTION satellite_geographic_position_at(
    LINE1 CHARACTER VARYING,
    LINE2 CHARACTER VARYING,
    AT_DATETIME TIMESTAMP WITHOUT TIME ZONE
) RETURNS GEOMETRY (POINTZ, 4326)
LANGUAGE PLPGSQL
AS $$
BEGIN
    RETURN ST_SetSRID(
        ST_GeomFromText(
            SATELLITE_GEOGRAPHIC_POSITION(
                LINE1,
                LINE2,
                AT_DATETIME
            )
        ), 4326
    );
END
$$;
```
