---
source_url: https://northflank.com/docs/v1/api/domains/delete-subdomain-path
title: Delete subdomain path | Domains | Northflank API docs
crawl_date: 2025-07-29T10:04:44.620873
watsonmd_version: 0.1.0
---

Domains / 

# Delete subdomain path

Delete a path.

Required permission

Account > SubdomainPaths > General > Update

Path parameters

    * domain

string required

Name of the domain

    * subdomain

string required

Name of the subdomain

    * subdomainPath

string required

Name of the path




Response body

  * {object}

Response object.

    * data

{object} required

Result data.




API

CLI

JS Client

DELETE /v1/domains/{domain}/subdomains/{subdomain}/paths/{subdomainPath}

Example response

200 OK

The operation was performed successfully.

JSON
    
    
    {
      "data": {}
    }