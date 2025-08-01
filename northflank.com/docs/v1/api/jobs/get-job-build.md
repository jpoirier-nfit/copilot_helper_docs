---
source_url: https://northflank.com/docs/v1/api/jobs/get-job-build
title: Get job build | Jobs | Northflank API docs
crawl_date: 2025-07-29T10:04:46.058391
watsonmd_version: 0.1.0
---

Jobs / 

# Get job build

Gets information about a build for the job

Required permission

Project > Jobs > Deployment > Deploy Build

Path parameters

    * projectId

string required

ID of the project

    * jobId

string required

ID of the job

    * buildId

string required

ID of the job build




Response body

  * {object}

Response object.

    * data

{object} required

Result data.

      * id

string required

ID of the build.

      * branch

string

Name of the branch the built commit belongs to.

      * pullRequestId

number

ID of the pull request the commit belongs to.

      * status

string

The status of the build.

one of

QUEUED, PENDING, UNSCHEDULABLE, STARTING, CLONING, BUILDING, UPLOADING, ABORTED, FAILURE, SUBMISSION_FAILURE, SUCCESS, CRASHED, IN_PROGRESS

      * sha

string

The sha of the built commit.

      * registry

{object}

        * uri

string

URI of that can be used to pull the image from the registry

      * concluded

boolean

Whether the build has finished.

      * createdAt

string

Timestamp of the build initiation.

      * success

boolean

Whether the build was successful.

      * message

string

Description of the build status.

      * buildConcludedAt

number

Timestamp of the build concluding.




API

CLI

JS Client

GET /v1/projects/{projectId}/jobs/{jobId}/build/{buildId}

Example response

200 OK

Returns data about the specified build.

JSON
    
    
    {
      "data": {
        "id": "joyous-view-6290",
        "branch": "main",
        "status": "SUCCESS",
        "sha": "12c15e7ee25fd78f567ebf87f9178b8ad70025b3",
        "concluded": true,
        "createdAt": "2021-07-28T15:55:38.296Z",
        "success": true,
        "message": "Image successfully built",
        "buildConcludedAt": 1606237973
      }
    }