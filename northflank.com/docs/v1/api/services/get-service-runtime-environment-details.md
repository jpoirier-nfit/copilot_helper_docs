---
source_url: https://northflank.com/docs/v1/api/services/get-service-runtime-environment-details
title: Get service runtime environment details | Services | Northflank API docs
crawl_date: 2025-07-29T10:04:44.244407
watsonmd_version: 0.1.0
---

Services / 

# Get service runtime environment details

Get details about the runtime environment accessible by the given service. Also requires the permission 'Project > Secrets > General > Read'

Required permission

Project > Secrets > Services > Read

Path parameters

    * projectId

string required

ID of the project

    * serviceId

string required

ID of the service




Response body

  * {object}

Response object.

    * data

{object} required

Result data.

      * runtimeEnvironment

{object} required

Details about all the secrets accessible by the service.

        * MY_VARIABLE_NAME

{object}

A stored secret and details about it and its value. This can have the name of any saved secret.

          * value

required

The value of the secret.

          * inheritedFrom

string

The ID of the secret group the secret is inherited from, if applicable.

          * addonId

string

The ID of the addon the secret is inherited from, if applicable.

          * priority

integer

The priority of the secret group the secret is inherited from, if applicable.

          * overriding

[array] required

An array containing data about other places the secret has been inherited from, but that are not being used as a secret with the same key exists with a higher priority.

            * {object}

Data about an overridden secret.

              * value

required

The value of the secret.

              * inheritedFrom

string required

The ID of the secret group the secret is inherited from.

              * addonId

string

The ID of the addon the secret is inherited from, if applicable.

              * priority

integer required

The priority of the secret group the secret is inherited from.

      * runtimeFiles

{object} required

Details about all the secrets accessible by the service.

        * /dir/fileName

{object}

A stored secret and details about it and its value. This can have the name of any saved secret.

          * value

{object} required

The value of the secret.

            * data

string

base64 encoded string of the file contents

            * encoding

string

Original encoding of the file

          * inheritedFrom

string

The ID of the secret group the secret is inherited from, if applicable.

          * priority

integer

The priority of the secret group the secret is inherited from, if applicable.

          * overriding

[array] required

An array containing data about other places the file has been inherited from, but that are not being used as a secret with the same file path exists with a higher priority.

            * {object}

Data about an overridden secret.

              * value

required

The value of the secret.

              * inheritedFrom

string required

The ID of the secret group the secret is inherited from.

              * priority

integer required

The priority of the secret group the secret is inherited from.




API

CLI

JS Client

GET /v1/projects/{projectId}/services/{serviceId}/runtime-environment/details

Example response

200 OK

Details of the runtime environment of the service.

JSON
    
    
    {
      "data": {
        "runtimeEnvironment": {
          "MY_VARIABLE_NAME": {
            "value": "abcdef123456",
            "inheritedFrom": "example-secret",
            "addonId": "example-addon",
            "priority": 10,
            "overriding": [
              {
                "value": "ffffffffffff",
                "inheritedFrom": "secret-2",
                "addonId": "addon-2",
                "priority": 0
              }
            ]
          }
        },
        "runtimeFiles": {
          "/dir/fileName": {
            "value": {
              "data": "VGhpcyBpcyBhbiBleGFtcGxlIHdpdGggYSB0ZW1wbGF0ZWQgJHtOT0RFX0VOVn0gdmFyaWFibGU=",
              "encoding": "utf-8"
            },
            "inheritedFrom": "example-secret",
            "priority": 10,
            "overriding": [
              {
                "value": "ffffffffffff",
                "inheritedFrom": "secret-2",
                "priority": 0
              }
            ]
          }
        }
      }
    }