# Linkage Agent Tools

Tools for the Childhood Obesity Data Initative (CODI) Linkage Agent to use to accept garbled input from data owners / partners, perform matching and generate network IDs. This can also be thought of as Semi-Trusted Third Party (STTP) tools.

These tool facilitate a Privacy Preserving Record Linkage (PPRL) process. They build on the open source [anonlink](https://github.com/data61/anonlink) software package.

## Installation

### anonlink-entity-service

The primary dependency of these tools is on the [anonlink-entity-service](https://anonlink-entity-service.readthedocs.io/en/stable/). This software package provides a web service for accessing [anonlink](https://github.com/data61/anonlink)'s matching capabilites. This software must be installed for the Linkage Agent Tools to work. Install instructions can be found on the [anonlink-entity-service Deployment page](https://anonlink-entity-service.readthedocs.io/en/stable/deployment.html)

After following the install instructions in the `anonlink-entity-service` documentation, you can confirm it is working if the API call to `/api/v1/status` responds as described in the example at https://anonlink-entity-service.readthedocs.io/en/stable/local-deployment.html to clarify when entity service is up and running correctly.

### MongoDB

Linkage Agent Tools uses [MongoDB](https://www.mongodb.com/) to store results obtained from the anonlink-entity-service. Install MongoDB by [downloading the community version](https://www.mongodb.com/try/download/community).

### Dependency Overview

Linkage Agent Tools is a set of scripts designed to interact with the previously mentioned anonlink-entity-service. They were created and tested on Python 3.7.4. The tools rely on two libraries: [Requests](https://requests.readthedocs.io/en/master/) and [pymongo](https://pymongo.readthedocs.io/en/stable/).

Requests is a library that makes HTTP requests. This is used for the tools to communicate with the web service offered by the anonlink-entity-service.

pymongo is a Python client library for MongoDB.

Linkage Agent Tools contains a test suite, which was created using [pytest](https://docs.pytest.org/en/latest/).

### Installing with an existing Python install

1. Download the tools as a zip file using the "Clone or download" button on GitHub.
1. Unzip the file.
1. From the unzipped directory run:

    `pip install -r requirements.txt`

### Installing with Anaconda

1. Install Anaconda by following the [install instructions](https://docs.anaconda.com/anaconda/install/).
    1. Depending on user account permissions, Anaconda may not install the latest version or may not be available to all users. If that is the case, try running `conda update -n base -c defaults conda`
1. Download the tools as a zip file using the "Clone or download" button on GitHub.
1. Unzip the file.
1. Open an Anaconda Powershell Prompt
1. Go to the unzipped directory
1. Run the following commands:
    1. `conda create --name codi`
    1. `conda activate codi`
    1. `conda install pip`
    1. `pip install -r requirements.txt`

## Configuration

Linkage Agent Tools is driven by a central configuration file, which is a JSON document. An example is shown below:

```
{
  "systems": ["site_a", "site_b", "site_c", "site_d", "site_e", "site_f"],
  "projects": ["name-sex-dob-phone", "name-sex-dob-zip",
    "name-sex-dob-parents", "name-sex-dob-addr"],
  "schema_folder": "/CODI/data-owner-tools/example-schema",
  "inbox_folder": "/CODI/inbox",
  "matching_results_folder": "/CODI/results",
  "output_folder": "/CODI/output",
  "entity_service_url": "http://localhost:8851/api/v1",
  "matching_threshold": 0.75,
  "mongo_uri": "localhost:27017",
  "blocked": false,
  "blocking_schema": "/CODI/data-owner-tools/example-schema/blocking-schema/lambda.json",
  "household_match": true,
  "household_schema": "/CODI/data-owner-tools/example-schema/household-schema/fn-phone-addr-zip.json"
}
```
A description of the properties in the file:
* **systems** - The set of data owners in this matching effort. These are short names for the participants. When data owners send zip files, it is expected that they will have the format of "data owner name".zip.
* **projects** - The anonlink [linkage projects](https://anonlink-entity-service.readthedocs.io/en/stable/tutorial/Record%20Linkage%20API.html#Create-Linkage-Project) that are going to be used in this matching effort. It assumes that the project names will have a corresponding anonlink schema file in the schema folder.
* **schema_folder** - A folder containing [anonlink schema files](https://clkhash.readthedocs.io/en/latest/schema.html). The schema files should be named "project name".json.
* **inbox_folder** - The folder where zip files recieved from data owners should be placed.
* **matching_results_folder** - Folder where the CSV containing the complete mapping of LINK_IDs to all data owners
* **output_folder** - Folder where CSV files are generated, one per data owner. These files contain LINK_IDs mapped to a single data owner.
* **entity_service_url** - The RESTful service endpoint for the anonlink-entity-service.
* **matching_threshold** - The threshold for considering a potential set of records a match when comparing in anonlink.
* **mongo_uri** - The URI to use when connecting to MongoDB to store or access results. For details on the URI structure, consult the [Connection String URI Format documentation](https://docs.mongodb.com/manual/reference/connection-string/)
* **blocked** - A boolean value for if the CLK's from the data owner in the inbox folder were generated via [blocking](https://anonlink-client.readthedocs.io/en/latest/tutorial/Blocking%20with%20Anonlink%20Entity%20Service.html)
* **blocking_schema** - The optional path to the file used by data owner tools for blocking
* **household_match** - A boolean true / false value for running the household pprl and matching options if household data was provided by the data owners
* **household_schema** - The path to the file used during household PPRL

## Input and Output Setup

Once you specify the paths outlined in the configuration section above, you need to put the zip files from each data owner into the `inbox_folder` specified, ex. with households from `systems` aka data owners `[site_a, site_b, site_c]`:
```
inbox/
  site_a.zip
  site_a_households.zip
  site_b.zip
  site_b_households.zip
  site_c.zip
  site_c_households.zip
  ...
```

After running the scripts in the order specified in the repository structure section below, the project will produce the following files in the `output_folder` specified in the config:
```
output/
  site_a.csv
  site_a_households.csv
  site_b.csv
  site_b_households.csv
  site_c.csv
  site_c_households.csv
  ...
```

## Structure

This project is a set of python scripts driven by a central configuration file, `config.json`. It is expected to operate in the following order:

1. Data owners transmit their garbled zip files to the Linkage Agent. These zip files should be placed into the configured inbox folder.
1. Run `validate.py` which will ensure all of the necessary files are present.
1. Run `drop.py` if you have done a previous matching run to clear old data in the database
1. When all data is present, run `match.py`, which will perform pairwise matching of the garbled information sent by the data owners. The matching information will be stored in MongoDB.
1. After matching, run `link_ids.py`, which will take all of the resulting matching information and use it to generate LINK_IDs, which are written to a CSV file in the configured results folder.
1. Once all LINK_IDs have been created, run `data_owner_ids.py` which will create a file per data owner that can be sent with only information on their LINK_IDs.

#### Example system folder hierarchy:

The `schema_folder` in the example below is using the example config paths from above. The schemas used by the data-owner during garbling of the data needs to be the same schemas pointed to in the linkage-agent `config.json`.

```
/CODI/
  linkage-agent-tools/
    ...
  inbox/
    site_a.zip
    site_a_households.zip
    site_a_block.zip
    site_b.zip
    site_b_households.zip
    site_b_block.zip
  output/
    site_a.csv
    site_a_households.csv
    site_b.csv
    site_b_households.csv
  data-owner-tools/
    ...
    example-schema/
      name-dob-ex.json
      name-phone-ex.json
      blocking-schema/
        lambda.json
      household-schema/
        fn-phone-addr-zip.json
```

## Running Tests

Linkage Agent Tools contains a unit test suite. Tests can be run with the following command:

`python -m pytest`

## [WIP] Jupyter Notebook

The `Linkage and Blocking Tuning Tool` Jupyter notebook is a work in progress meant for testing and tuning different configurations against the synthetic data set with Data Owner Tools and Linkage Agent Tools projects running on the same machine. It will currently run all necessary scripts to do end to end testing of the entire PPRL process but is still being improved and will include more documentation when finalized.

## Notice

Copyright 2020 The MITRE Corporation.

Approved for Public Release; Distribution Unlimited. Case Number 19-2008
