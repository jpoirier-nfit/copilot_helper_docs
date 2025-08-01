---
source_url: https://northflank.com/docs/v1/api/jobs/get-job-build-arguments
title: Get job build arguments | Jobs | Northflank API docs
crawl_date: 2025-07-29T10:04:45.863385
watsonmd_version: 0.1.0
---

Jobs / 

# Get job build arguments

Gets the build arguments of the given job. If the API key does not have the permission 'Project > Secrets > General > Read', secrets inherited from secret groups will not be displayed.

Required permission

Project > Secrets > Jobs > Read

Path parameters

    * projectId

string required

ID of the project

    * jobId

string required

ID of the job




Query parameters

    * show

string

Which secrets to display - if set to `this` only the group's secrets are displayed, if set to `inherited` only secrets from linked addons are displayed, if set to `all` or not provided, both are displayed.

one of

this, inherited, all

    * replaceTemplatedValues

string

If templated secrets should be replaced with their inferred value rather than returned as template strings.

one of

true




Response body

  * {object}

Response object.

    * data

{object} required

Result data.

      * buildArguments

{object} required

The build arguments, formatted as a JSON object. If the `show` parameter is set to `this`, this will only contain secrets saved to this entity. If the `show` parameter is set to `inherited`, this will only contain secrets inherited from linked secret groups. Otherwise, this will contain both.

      * buildFiles

{object} required

The build secret files, formatted as a JSON object. If the `show` parameter is set to `this`, this will only contain files saved to this entity. If the `show` parameter is set to `inherited`, this will only contain files inherited from linked secret groups. Otherwise, this will contain both.




API

CLI

JS Client

GET /v1/projects/{projectId}/jobs/{jobId}/build-arguments

Example response

200 OK

The build arguments for the job.

JSON
    
    
    {
      "data": {
        "buildArguments": {
          "ARGUMENT_1": "abcdef",
          "ARGUMENT_2": "12345"
        },
        "buildFiles": {
          "/dir/fileName": {
            "data": "VGhpcyBpcyBhbiBleGFtcGxlIHdpdGggYSB0ZW1wbGF0ZWQgJHtOT0RFX0VOVn0gdmFyaWFibGU=",
            "encoding": "utf-8"
          }
        }
      }
    }