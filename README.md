# CMS Utilization & Payment Database

### The purpose of the project is to provide a containerized database service loaded with CMS data in order to increase the utility of the dataset and reduce the technical overhead associated with consuming it.

Stakeholders pursuing healthcare price transparency can use this database to learn about costs & volumes of HCPCS services, provider performance (as reported in source data and additional derived analytic data), and understand finer details related to healthcare costs.

Examples of detailed healthcare costs:
- variance between charges & payments
- variance in payments between providers for the same service
- average costs nationally for individual healthcare services

Examples of derived analytic data:
- Ranking of Providers by Total Services Provided
- Ranking of Providers by HCPCS Services Provided
- Estimated Sum Amount `charged` & `received` by Provider
- Estimated Sum Amount `charged` & `received` by Provider & HCPCS Service
- Estimated Sum Amount `charged` & `received` by HCPCS Service
- Ranking of Providers by Total $ Charged to CMS
- Ranking of Providers by Total $ Paid from CMS

For background knowledge see :
- [Source Data: CMS Utilization & Payment Data](https://data.cms.gov/Medicare-Physician-Supplier/Medicare-Provider-Utilization-and-Payment-Data-Phy/utc4-f9xp/data)
- [NPPES NPI: wikipedia](https://en.wikipedia.org/wiki/National_Provider_Identifier)
- [NPPES NPI: api](https://npiregistry.cms.hhs.gov/)
- [HCPCS: wikipedia](https://en.wikipedia.org/wiki/Healthcare_Common_Procedure_Coding_System)
- [HCPCS: CMS Datasets](https://www.cms.gov/Medicare/Coding/HCPCSReleaseCodeSets/Alpha-Numeric-HCPCS.html)

---


##  <a id="table-of-contents"></a>Table of Contents

* [Quick Start](#quick-start)
* [Start with Prisma for GraphQL API](#start-with-prisma)
* [Source data](#db-data-source)
* [Schema](#db-schema)
  * [Source Data Tables](#db-schema-tables-source)
  * [Analytic Data Tables](#db-schema-tables-analytic)
* [Prisma IDs](#prisma-requirements)
* [GraphQL Queries](#graphql-queries)
  * [HCPCS Service Information & Aggregate Performance](#graphql-hcpcs-service-performance)
  * [Provider Identity & Performance](#graphql-provider-id-and-performance)
* [Development](#development)
  * [Run Service](#dev-run-Service)
  * [Container Maintenance](#dev-container-maintenance)
  * [Generate DB Export](#dev-generate-db-export)
  * [Restore DB from Export](#dev-restore-db-from-export)
  * [Build & Load DB via ETL Process](#dev-full-etl-process)
  * [Container Entry Shortcut](#dev-container-entry-shortcut)
* [Project Origin & Inspiration](#project-origin)
* [Additional Data Sets](#additional-data-sets)
* [Versioning](#versioning)
* [License](#license)


---

##  <a id="quick-start"></a>Quick Start

```
make start
```
- Postgres Database | port 5432
- Credentials
	- **database:** govdata
	- **user:** dbuser
	- **password:** dbpassword

This command starts the database and restores the schema (structure & data) from a db.dump that is baked into the docker image.

This initiation process will take 10+ mins (sample `time` report provided below) to complete on the first run. Subsequent starts rely on persistent data.

	:: real	11m5.184s
	:: user	0m15.680s
	:: sys	0m2.800s
---

##  <a id="start-with-prisma"></a>Start with Prisma for GraphQL API

#### Use this command OR `make start`. Not Both.

```
# get host ip address # this is the ip the prisma container will call to reach the db containers
ifconfig | grep inet

# update PRISMA_CONFIG in docker-compose.prisma.yml
# by replacing __REPLACE__WITH__LOCAL__IP__ with ip address from above

# start db & prisma service
make start-with-prisma

# ensure prisma is installed globally
sudo npm i prisma

# deploy prisma config to prisma server
npx prisma deploy
```
- Postgres Database | port 5432
- [GraphQL Playground](http://0.0.0.0:4466/) | port 4466

This command starts the database and starts a prisma service that provides a GraphQL API to the data.

#### 

**Note 1:** Use Node v10.

**Note 2:** Before you can successfully deploy the prisma services, you must update the host defination inside the `PRISMA_CONFIG` environment variable stored in the [`docker-compose.prisma.yml`](./docker-compose.prisma.yml). To accomplish this, simply replace the placeholder `__REPLACE__WITH__LOCAL__IP__` (near line 18) with the IP of your local machine so that the prisma container can reach the DB container.

---

##  <a id="db-data-source"></a>Source data

The foundation of this project is a `.csv` dataset titled, _Medicare Provider Utilization and Payment Data: Physician and Other Supplier PUF CY2016_.

You can find a full description of the dataset on [their site](https://data.cms.gov/Medicare-Physician-Supplier/Medicare-Provider-Utilization-and-Payment-Data-Phy/utc4-f9xp/data), but for convenience, a summarized description is below: 

> The Centers for Medicare & Medicaid Services (CMS) has prepared a public data set that provides information on services and procedures provided to Medicare beneficiaries by physicians and other healthcare professionals.
The dataset contains information on utilization, payment (allowed amount and Medicare payment), and submitted charges organized by National Provider Identifier (NPI), Healthcare Common Procedure Coding System (HCPCS) code, and place of service.
This data represents calendar year 2016 and contains 100% final-action physician/supplier Part B non-institutional line items for the Medicare fee-for-service population.

---

##  <a id="db-schema"></a>Schema

Within the database, the data is organized into 9 tables (5 **source data** tables & 4 **analytic data** tables).

### <a id="db-schema-tables-source"></a>Source Data Tables

![cms-schema source-data-tables](assets/img/readme/cms_schema_source_data.png "CMS Schema | Tables based on .CSV Source Data")

- `services` contains all HCPCS Services represented in the source file.

- `providers` (and related tables `providers_individuals` & `providers_organizations`) contains identity information & address for each provider (distinguished by `providers.entity_type`).

- `provider_performance` contains performance statistics for each HCPCS service provided by each provider.

### <a id="db-schema-tables-analytic"></a>Analytic Data Tables

![cms-schema analytic-data-tables](assets/img/readme/cms_schema_analytic_data.png "CMS Schema | Tables based on .CSV Source Data")

- `service_performance` aggregates performance statistics for each HCPCS service across all providers.

- `service_provider_performance` extends the data contained in `provider_performance`, estimates cumulative amounts charged & paid (`n_of_svcs` * `amt`), and also generates performance rankings by compares each provider's performance attributes against **all other providers** that also perform the service.

- `service_provider_performance_summary` aggregates each provider's performance statistics across all HCPCS services. The summary also estimates cumulative amounts charged & paid (`n_of_svcs` * `amt`) and generates performance rankings by comparing each provider's performance attributes against **all other providers**.

- `service_provider_performance_summary_type` is used to define different ways of generating summaries (provider groupings and/or service subsets) within `service_provider_performance_summary`.
  
	Currently summaries supported:
    - all providers + all services
    - all providers + non-drug HSPCS services
    - all providers + drug HSPCS services
    - [**planned feature**](https://github.com/sudowing/cms-utilization-db/issues/1): enterprise providers + all services

---

##  <a id="prisma-requirements"></a>Prisma IDs

The first planned consumer of this DB was a GraphQL service.

[That feature](#start-with-prisma) uses [prisma](https://github.com/prisma/prisma) to provide a GraphQL API to the data. Unfortunately, when developing that feature, I noticed that prisma currently doesn't support composite keys (although the feature is proposed for [Datamodel v2](https://github.com/prisma/prisma/issues/3405)).

To resolve this in the meantime, new **auto incrementing** primary keys were defined (as `prisma_id`). The original keys still exist, but are no long the primary key for their respective tables.

---

##  <a id="graphql-queries"></a>GraphQL Queries

GraphQL queries can be pretty overwhelming to the newly initiated.

In order to speed adoption (and ease developers into the data), I've built out a few GraphQL queries and saved them on [GraphQL Bin](https://www.graphqlbin.com/v2/new) -- which is like a pastebin for these kinds of things.

I'll add to this list regularly, so be sure to check back. Also, for anyone wanting to contribute... this is some low hanging fruit.

### GraphQL Bin Stored Queries

- <a id="graphql-hcpcs-service-performance"></a>[HCPCS Service Information & Aggregate Performance](https://graphqlbin.com/v2/6XZLfA)
- <a id="graphql-provider-id-and-performance"></a>[Provider Identity & Performance](https://graphqlbin.com/v2/jZpRfl)

---

##  <a id="development"></a>Development

### <a id="dev-run-Service"></a>Run Service

```
make run
```

This command starts the DB service, and mounts several of the local directories (scripts, sql, tmp) in the container.

### <a id="dev-container-maintenance"></a>Container Maintenance

```
# @docker-compose stop
make stop
	
# @docker-compose down --remove-orphan
make clean
```

These commands stop all containers & remove all containers.


### <a id="dev-generate-db-export"></a>Generate DB Export

```
make db-dump-sql
```

This command connects to the container running the DB service and generates a sql export via `pg_dump`.

**NOTE 1:** This sql.dump file is large (1gb) and as such, is not stored in the repo (`.gitignore`). To generate this file (for inspection or when publishing a new docker image), 
you must generate this file, and have it locally, **before** baking the docker image.

### <a id="dev-restore-db-from-export"></a>Restore DB from Export

```
make db-load-via-sql
```

This command connects to the container running the DB service and restores the db using the sql.dump file generated above.

### <a id="dev-full-etl-process"></a>Build & Load DB via ETL Process

```
make db-load-via-etl
```

This command builds the database through an ETL process that downloads the original `.csv`, loads it into temp tables, then builds prod tables and populates them from the temp tables. 

Anyone interested in validating the integrity of the data provided by the published docker image for project, should focus their attention here.

**NOTE 1:** ETL process takes 2.5+ hours.

**NOTE 2:** ETL process downloads the original CSV from the government’s servers and stores it inside the container at `/tmp/cms-utilization.csv`
If you're running the project in development mode (`make run`), this file should be available to you at `/volumes/tmp/cms-utilization.csv` 

### <a id="dev-container-entry-shortcut"></a>Container Entry Shortcut

```
make enter
```

These command is a shortcut for entering the running `cms-utilization-db_cms-utilization-postgres_1` container. You're welcome.


---

##  <a id="project-origin"></a>Project Origin & Inspiration

I've been interested in healthcare utilization & payment data since I read an article by Steven Brill in _Time Magazine_ in 2013.

In the years since, I've seen dozens of smaller, similar reports in various media, but I've yet to see anything as comprehensive and detailed as the original piece.

In appreciation of Brill's work, and to direct interested parties towards information that is probably relevant to their projects, I'm publishing links to some of those resources here.

- [Original Time Article](http://time.com/198/bitter-pill-why-medical-bills-are-killing-us/)
  - **Title**: Bitter Pill: Why Medical Bills Are Killing Us
  - **Author**: Steven Brill
  - **Date**: Apr 04, 2013
- [Minutes from Congressional Hearing](https://www.govinfo.gov/content/pkg/CHRG-113shrg87496/pdf/CHRG-113shrg87496.pdf)
  - **Event**: Senate Hearing before the Committee on Finance
  - **Title**: High Prices, Low Transparency: The Bitter Pill of Health Care Costs
  - **Date**: June 18, 2013
  - **Note**: Specific attention to Brill's the opening statement on page 5-6 (pdf page 9-10).
- [Federal Register Volume 83, Number 192](https://www.govinfo.gov/content/pkg/FR-2018-10-03/html/2018-21500.htm)
  - **Section X**: Requirements for Hospitals To Make Public a List of Their 
  Standard Charges via the Internet
- [CMS FAQs about 2019 Pricing Transparency Requirements](https://www.cms.gov/Medicare/Medicare-Fee-for-Service-Payment/AcuteInpatientPPS/Downloads/FAQs-Req-Hospital-Public-List-Standard-Charges.pdf)


---

##  <a id="additional-data-sets"></a>Additional Data Sets

Similar projects can be built using data from these additional data stores:
- [CMS Data Catalog](https://data.cms.gov/)
- [Open Payments: Overview](https://www.cms.gov/openpayments/)
- [Open Payments: Data](https://openpaymentsdata.cms.gov/)
- [Medicare.gov: Data](https://data.medicare.gov/)
- [Medicaid & CHIP Open Data](https://data.medicaid.gov/)
- [DATA.GOV: Health Data Catalog](https://catalog.data.gov/dataset?_organization_limit=0&organization=hhs-gov#topic=health_navigation)
- [HealthData.gov](https://healthdata.gov/)
- [Chronic Conditions Data Warehouse](https://www.ccwdata.org/web/guest/home)

---

##  <a id="versioning"></a>Versioning

We use [SemVer](http://semver.org/) for versioning. For the versions available, see the [tags on this repository](https://github.com/sudowing/cms-utilization-db/tags). 

---

##  <a id="license"></a>License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details
