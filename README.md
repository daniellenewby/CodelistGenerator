
<!-- README.md is generated from README.Rmd. Please edit that file -->

<!-- badges: start -->

[![codecov.io](https://codecov.io/github/oxford-pharmacoepi/CodelistGenerator/coverage.svg?branch=main)](https://codecov.io/github/oxford-pharmacoepi/CodelistGenerator?branch=main)
[![R-CMD-check](https://github.com/oxford-pharmacoepi/CodelistGenerator/workflows/R-CMD-check/badge.svg)](https://github.com/oxford-pharmacoepi/CodelistGenerator/actions)
<!-- badges: end -->

# CodelistGenerator

## Introduction

CodelistGenerator is used to create a candidate set of codes for helping
to define patient cohorts in data mapped to the OMOP common data model.
A little like the process for a systematic review, the idea is that for
a specified search strategy, CodelistGenerator will identify a set of
concepts that may be relevant, with these then being screened to remove
any irrelevant codes.

## Installation

You can install the development version of CodelistGenerator like so:

``` r
install.packages("remotes")
remotes::install_github("oxford-pharmacoepi/CodelistGenerator")
```

## Connecting to the OMOP CDM vocabularies

### Option 1: Connect to a live OMOP CDM database

``` r
# example with postgres database connection details
server_dbi<-Sys.getenv("server")
user<-Sys.getenv("user")
password<- Sys.getenv("password")
port<-Sys.getenv("port")
host<-Sys.getenv("host")

db <- DBI::dbConnect(RPostgres::Postgres(),
                dbname = server_dbi,
                port = port,
                host = host,
                user = user,
                password = password)

# name of vocabulary schema
vocabulary_database_schema<-Sys.getenv("vocabulary_schema")
```

### Option 2: Download the vocabularies from Athena

You will first need to obtain the OMOP CDM vocabularies from
<https://athena.ohdsi.org>. Once these are downloaded, you can make a
vocabulary only SQLite database like so:

``` r
vocab.folder<-Sys.getenv("omop_cdm_vocab_path") # path to directory of unzipped files
concept<-read_delim(paste0(vocab.folder,"/CONCEPT.csv"),
     "\t", escape_double = FALSE, trim_ws = TRUE)
concept_relationship<-read_delim(paste0(vocab.folder,"/CONCEPT_RELATIONSHIP.csv"),
     "\t", escape_double = FALSE, trim_ws = TRUE) 
concept_ancestor<-read_delim(paste0(vocab.folder,"/CONCEPT_ANCESTOR.csv"),
     "\t", escape_double = FALSE, trim_ws = TRUE)
concept_synonym<-read_delim(paste0(vocab.folder,"/CONCEPT_SYNONYM.csv"),
     "\t", escape_double = FALSE, trim_ws = TRUE)
vocabulary<-read_delim(paste0(vocab.folder,"/VOCABULARY.csv"),
     "\t", escape_double = FALSE, trim_ws = TRUE)

db <- dbConnect(RSQLite::SQLite(), ":memory:")
dbWriteTable(db, "concept", concept, overwrite=TRUE)
dbWriteTable(db, "concept_relationship", concept_relationship, overwrite=TRUE)
dbWriteTable(db, "concept_ancestor", concept_ancestor, overwrite=TRUE)
dbWriteTable(db, "concept_synonym", concept_synonym, overwrite=TRUE)
dbWriteTable(db, "vocabulary", vocabulary)
rm(concept,concept_relationship, concept_ancestor, concept_synonym, vocabulary)

vocabulary_database_schema<-"main"
```

### Option 3: Use Eunomia (for testing and examples only - Eunomia does not include a full set of vocabularies)

``` r
library(CodelistGenerator)
library(Eunomia)
library(RSQLite)
library(DBI)
untar(xzfile(system.file("sqlite", "cdm.tar.xz", package = "Eunomia"), open = "rb"),
        exdir =  tempdir())
db <- DBI::dbConnect(RSQLite::SQLite(), paste0(tempdir(),"\\cdm.sqlite"))
```

## Example search using Eunomia

Every codelist is specific to a version of the OMOP CDM vocabularies, so
we can first check the version.

``` r
get_vocab_version(db=db,
                  vocabulary_database_schema = "main")
#> [1] "v5.0 18-JAN-19"
```

We can then search for asthma like so

``` r
get_candidate_codes(keywords="asthma",
                    domains = "Condition",
                    db=db,
                    vocabulary_database_schema = "main")
#> # A tibble: 2 x 4
#>   concept_id concept_name     domain_id vocabulary_id
#>        <dbl> <chr>            <chr>     <chr>        
#> 1    4051466 Childhood asthma Condition SNOMED       
#> 2     317009 Asthma           Condition SNOMED
```

Perhaps we want to exclude asthma in children as part of the search
strategy, in which case this can be added like so

``` r
get_candidate_codes(keywords="asthma",
                    domains = "Condition",
                    exclude = "Childhood asthma",
                    db=db,
                    vocabulary_database_schema = "main")
#> # A tibble: 1 x 4
#>   concept_id concept_name domain_id vocabulary_id
#>        <dbl> <chr>        <chr>     <chr>        
#> 1     317009 Asthma       Condition SNOMED
```

Please see vignettes for further details.

## Development status

Alpha
