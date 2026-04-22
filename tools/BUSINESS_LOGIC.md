# Business Logic Rules

This document defines the business logic rules for evaluating vehicle features based on the parameters defined in the project's artifact data.

## Data Source

All business logic and feature criteria are derived from the following source:

* **File Path**: `artifacts\params.csv`
* **Purpose**: Serves as the master reference for feature definitions, categorization, and tiering criteria.

## Parameter Data Structure

The business logic utilizes the following column structure from the reference data:

| Column Name                  | Description                                                      |
| :--------------------------- | :--------------------------------------------------------------- |
| **Domain / Category**        | The high-level functional area (e.g., EV, Infotainment, ADAS).   |
| **Parameter**                | The specific feature or technology being evaluated.              |
| **Category Description**     | A high-level overview of the functional domain.                  |
| **Parameter Description**    | A detailed explanation of what the specific parameter covers.    |
| **Basic — Criterion**        | Minimum requirements to meet the 'Basic' implementation level.   |
| **Medium — Criterion**       | Requirements to meet the 'Medium' implementation level.          |
| **High — Criterion**         | Requirements to meet the 'High' implementation level (Advanced). |
| **Optimised Parameter Name** | The standardized name used for reporting and analysis.           |

##
