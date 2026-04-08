# Compilation and Running Guide

This document explains how to compile and run all parts of the NoiseModelling project.

---

## Prerequisites

| Tool | Minimum Version | Notes |
|------|----------------|-------|
| Java (JDK) | 11 | Java 11 or later required; tested with OpenJDK 17 |
| Apache Maven | 3.6+ | Used for the core library modules; tested with Maven 3.9.9 |
| Gradle | 7+ (or use the included wrapper) | Used for the `wps_scripts` command-line application |
| Docker | any recent version | Optional – required only for PostGIS integration tests |

Verify your installation:

```bash
java -version
mvn -version
gradle --version   # or: ./gradlew --version inside wps_scripts/
```

---

## Repository Structure

```
NoiseModelling_Canada/
├── noisemodelling-emission/      # Sound emission library (Maven module)
├── noisemodelling-pathfinder/    # Noise path-finding library (Maven module)
├── noisemodelling-propagation/   # Noise propagation library (Maven module)
├── noisemodelling-jdbc/          # Database connectivity library (Maven module)
├── noisemodelling-tutorial-01/   # Tutorial / demo application (Maven module)
├── wps_scripts/                  # Standalone command-line WPS runner (Gradle)
├── wpsbuilder/                   # Web-based WPS workflow builder (static HTML)
└── pom.xml                       # Parent Maven POM (aggregates all Maven modules)
```

---

## 1. Core Java Libraries (Maven)

The four core libraries (`noisemodelling-emission`, `noisemodelling-pathfinder`,
`noisemodelling-propagation`, `noisemodelling-jdbc`) and the tutorial application
are all Maven modules managed by the root `pom.xml`.

### 1.1 Build All Modules (skip tests)

From the repository root:

```bash
mvn clean install -DskipTests
```

Compiled JARs are placed in each module's `target/` directory and installed into
the local Maven repository (`~/.m2`), making them available to other modules and
to the `wps_scripts` Gradle build.

### 1.2 Build and Run All Tests

```bash
mvn clean test
```

Most tests use an embedded H2GIS database and need no external services.

### 1.3 Build a Single Module

```bash
# Example: build only the emission library
mvn clean install -DskipTests -pl noisemodelling-emission
```

### 1.4 Build with Javadoc and Source JARs

```bash
mvn clean verify -DskipTests
```

---

## 2. Tutorial Application (`noisemodelling-tutorial-01`)

This module demonstrates Lday / Levening / Lnight / Lden noise-level computation
from road-traffic data.

### 2.1 Build

```bash
mvn clean package -pl noisemodelling-tutorial-01 -am -DskipTests
```

The `-am` flag also builds upstream dependencies automatically.

### 2.2 Run via Maven

```bash
mvn exec:java -pl noisemodelling-tutorial-01 \
    -Dexec.mainClass=org.noise_planet.nmtutorial01.Main
```

### 2.3 Run as a Stand-alone JAR

```bash
java -cp noisemodelling-tutorial-01/target/noisemodelling-tutorial-01-*-jar-with-dependencies.jar \
    org.noise_planet.nmtutorial01.Main
```

Results (CSV files) are written to `noisemodelling-tutorial-01/target/`.

### 2.4 Run Tests with a PostGIS Database (optional)

Start a PostGIS container:

```bash
docker run -d --name noisemodelling-postgres \
    -p 5432:5432 \
    -e POSTGRES_USER=noisemodelling \
    -e POSTGRES_PASSWORD=noisemodelling \
    -e POSTGRES_DB=noisemodelling_db \
    --health-cmd='pg_isready' \
    --health-interval=10s \
    --health-timeout=5s \
    --health-retries=5 \
    postgis/postgis:16-3.4
```

Then run the module tests:

```bash
mvn clean test -pl noisemodelling-tutorial-01 -am
```

Stop and remove the container when finished:

```bash
docker stop noisemodelling-postgres
docker rm noisemodelling-postgres
```

---

## 3. WPS Scripts – Standalone Command-Line Application (`wps_scripts/`)

This Gradle project produces a self-contained command-line runner for Groovy WPS
scripts. It depends on the core Maven libraries, so **build those first** (see
[Section 1.1](#11-build-all-modules-skip-tests)).

### 3.1 Build

```bash
cd wps_scripts
./gradlew build
```

> **Windows:** use `gradlew.bat build`

The distribution archive is created at:

```
wps_scripts/build/distributions/NoiseModelling_without_gui-<version>.zip
```

### 3.2 Prepare the Runtime

Extract the distribution archive:

```bash
cd wps_scripts/build/distributions
unzip NoiseModelling_without_gui-*.zip
cd NoiseModelling_without_gui-*
```

### 3.3 Run the Get-Started Tutorial

The extracted directory contains a ready-to-run script:

**Linux / macOS:**

```bash
./get_started_tutorial.sh
```

**Windows:**

```bat
get_started_tutorial.bat
```

These scripts perform the full workflow described in the
[NoiseModelling Get Started Tutorial](https://noisemodelling.readthedocs.io/en/latest/Get_Started_Tutorial.html):

1. Import shapefiles / GeoJSON into the embedded database.
2. Run the noise-level-from-traffic calculation.
3. Export the results to a shapefile.

### 3.4 Run Individual WPS Scripts Manually

```bash
./bin/wps_scripts -w ./ \
    -s noisemodelling/wps/Import_and_Export/Import_File.groovy \
    -pathFile resources/org/noise_planet/noisemodelling/wps/buildings.shp
```

> **Windows:** replace `./bin/wps_scripts` with `bin\wps_scripts.bat`

### 3.5 Sync the Gradle Version with the Maven POM (optional)

If you update the version in `pom.xml`, run:

```bash
cd wps_scripts
./update_gradle_version.sh
```

---

## 4. WPS Builder (`wpsbuilder/`)

The WPS Builder is a static web application for visually composing WPS workflows.
It requires no compilation step.

### 4.1 Serve Locally

Serve the `wpsbuilder/` directory with any HTTP server. For example, using Python:

```bash
# Python 3
cd wpsbuilder
python3 -m http.server 8080
```

Then open <http://localhost:8080> in your browser.

> **Note:** The WPS Builder connects to a GeoServer instance at `/geoserver/ows`
> to discover available WPS processes. When running locally without GeoServer, the
> process palette will be empty, but the UI layout and XML editor are still
> functional.

---

## 5. Quick Reference

| Goal | Command |
|------|---------|
| Build all Maven modules (no tests) | `mvn clean install -DskipTests` |
| Run all Maven tests | `mvn clean test` |
| Build a single Maven module | `mvn clean install -DskipTests -pl <module-name>` |
| Build WPS scripts distribution | `cd wps_scripts && ./gradlew build` |
| Run tutorial-01 | `mvn exec:java -pl noisemodelling-tutorial-01 -Dexec.mainClass=org.noise_planet.nmtutorial01.Main` |
| Serve WPS Builder | `cd wpsbuilder && python3 -m http.server 8080` |

---

## 6. Troubleshooting

- **`java.lang.UnsupportedClassVersionError`** – your runtime JDK is older than
  11. Install JDK 11 or later.
- **Gradle build fails with dependency resolution errors** – ensure you have
  internet access and that the Maven local repository contains the core
  NoiseModelling JARs (run `mvn clean install -DskipTests` from the root first).
- **PostGIS tests fail** – confirm the Docker container is healthy
  (`docker ps --filter name=noisemodelling-postgres`) before running `mvn test`.
- **`maven-gpg-plugin` errors during build** – the GPG signing step only runs
  during `deploy`. Use `mvn clean install` (not `deploy`) for local builds.
