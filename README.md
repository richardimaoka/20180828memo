Uses Akka typed, Cluster, HTTP, Persistence and Kubernetes

## TODO as of 20180822

- sbt project = common //not sure if we should do analysis in this directory
- sbt project = check-digit //almost done in terms of analysis, but text to be brushed up
- sbt project = hmda-platform (/hmda directory) //almost done in terms of analysis, but text to be brushed up
- sbt project = institutional-api //almost done in terms of analysis, but text to be brushed up
- Kubernetes stuff if there is enough time for this... not sure as I'm not an expert on this although it is an area of interest for me

## sbt project = common

## sbt project = check-digit

According to the [doc](https://github.com/cfpb/hmda-platform/tree/master/docs/v2#check-digit), it does:

> Microservice that exposes functionality to create a check digit from a loan id, and to validate `Univeral Loan Identifiers`

The [uli API](https://github.com/cfpb/hmda-platform/blob/master/docs/v2/api/uli.md) page already gives the API.
The route is defined in `ULIHttpApi` [(source code)](https://github.com/cfpb/hmda-platform/blob/master/check-digit/src/main/scala/hmda/uli/api/http/ULIHttpApi.scala)

This is not complicated internally, as it currently defines the validation logic in `validation.ULI` but no database or external service are used.

`HdmaUli` is the entry point of this project, which starts up an instance of `HmdaUliApi` actor, and the the actor spins up an HTTP server
as specified in [uli API](https://github.com/cfpb/hmda-platform/blob/master/docs/v2/api/uli.md).

- `POST: uli/checkDigit`: Calculates check digit and full ULI from a loan id.
   -  the validation & check-digit generation logic is defined in `ULI.checkDigit`
   - **memo (richard):** but is this complete? It only checks String length of the loan ID, if it is alpha-numeric, then generate the check digit
- `POST: uli/checkDigit`: You can also upload a file as multipart request
   - processes the file as Akka Stream `Source` using `formData.parts` within the `fileUpload` directive
   - use the same logic - `ULI.checkDigit`
- `POST: uli/checkDigit/csv`: Upload CSV`: similar to uploading a file
- `POST: uli/validate`: Upload CSV`: post a single-field JSON `{ "uli", "...." }`, 
   - the validation logic is in `ULI.validateULI`
- `POST: uli/validate`: Upload CSV`: You can also upload a file as multipart request
- `POST: uli/validate/csv`: Upload CSV`: similar to uploading a file
  
  
## sbt project = hmda-platform (/hmda directory)

Main = `object HmdaPlatform`

- It initializes Akka Cluster, and spawns four typed Actors
  - `HmdaPersistence` [(source code)](https://github.com/cfpb/hmda-platform/blob/master/hmda/src/main/scala/hmda/persistence/HmdaPersistence.scala)
  - `HmdaValidation` [(source code)](https://github.com/cfpb/hmda-platform/blob/master/hmda/src/main/scala/hmda/validation/HmdaValidation.scala)
  - `HmdaPublication` [(source code)](https://github.com/cfpb/hmda-platform/blob/master/hmda/src/main/scala/hmda/publication/HmdaPublication.scala)
  - `HmdaApi` [(source code)](https://github.com/cfpb/hmda-platform/blob/master/hmda/src/main/scala/hmda/api/http/HmdaApi.scala)

### HmdaPersistence

It `spawns` a sharded entity with `InstitutionPersistence.behavior` ([source code](https://github.com/cfpb/hmda-platform/blob/master/hmda/src/main/scala/hmda/persistence/HmdaPersistence.scala#L50)), which will be used later by `HmdaAdminApi`

```scala
sharding.spawn(
  behavior = entityId => InstitutionPersistence.behavior(entityId),
  Props.empty,
  typeKey,
  ClusterShardingSettings(system),
  maxNumberOfShards = shardNumber,
  handOffStopMessage = InstitutionStop
)
```

### HmdaValidation

**memo (richard):** It seems like it is in progress? Looking at the implementation, it is not doing almost anything.

### HmdaPublication

**memo (richard):** It seems like it is in progress? Looking at the implementation, it is not doing almost anything.

### HmdaApi

Serves (part of) the API described in (except the institutinal api) https://github.com/cfpb/hmda-platform/blob/master/docs/v2/api/public-api.md

This starts up four different HTTP servers, at four different host & port combinations.
By default in the dev environment, it uses localhost and the following ports:

- hmda-filing-api - localhost:8080
- hmda-admin-api  - localhost:8081
- hmda-public-api - localhost:8082
- hmda-ws-api     - localhost:9080

Each HTTP server is started with `Http().bindAndHandle()` inside an Actor as described below:

#### hmda-filing-api - `HmdaFilingApi` Actor

This supports the following HTTP methods and endpoint:

- `GET / ` : it just returns the following JSON
- `OPTIONS / ` : returns the `"OPTIONS": String` as the HTTP response body

```
{
  status: "OK",
  service: "hmda-filing-api",
  time: "2018-08-21T14:47:29.646Z",
  host: "matsukaze-PC"
}
```

**memo (richard):** It seems like work-in-progress, as this HTTP server doesn't do almost anything.

####  hmda-admin-api `HmdaAdminApi` Actor

- https://github.com/cfpb/hmda-platform/blob/master/docs/v2/api/admin-api.md#hmda-platform-admin-api

hmda-admin-api HTTP server returns a similar json as hmda-filing-api at /, additionally supports following endpoints:

- `GET / ` : it just returns a JSON similar to hmda-filing-api
- `OPTIONS / ` : returns the  "OPTIONS": String` as the HTTP response body, same as hmda-filing-api
- `GET:    /institutions/${LEI}` : get the institution information for `${LEI}`
  - Sends `GetInstitution`command to `institutionPersistence` ActorRef, then waits for `Some` which contains the institution information
- `POST:   /institutions/` :  accepts JSON represented by case class` Institution`([source code](https://github.com/cfpb/hmda-platform/blob/master/common/src/main/scala/hmda/model/institution/Institution.scala))
  - Sends `CreateInstitution` command to `institutionPersistence` ActorRef, then waits for `InstitutionCreated`(HTTP 201 Created) event
- `PUT:    /institutions/` : accepts JSON represented by case class` Institution`([source code](https://github.com/cfpb/hmda-platform/blob/master/common/src/main/scala/hmda/model/institution/Institution.scala))
  - Sends `ModifyInstitution` command to `institutionPersistence` ActorRef, then waits for `InstitutionModified`(HTTP 202 Accepted) or `InstitutionNotExists`(HTTP 404 Not Found) event
- `DELETE: /institutions/` :   accepts JSON represented by case class` Institution`([source code](https://github.com/cfpb/hmda-platform/blob/master/common/src/main/scala/hmda/model/institution/Institution.scala))
  - Sends `DeleteInstitution` command to `institutionPersistence` ActorRef, then waits for `InstitutionDeleted`(HTTP 202 Accepted) or `InstitutionNotExists`(HTTP 404 Not Found) event

`HmdaAdminApi` is a persistent actor, to achieve the event-sourced style architecture. As described above, commands are sent from the above `/institutions/` endpoint,
then the corresponding persistent actor for the specified entity id receives the command and persist the event to the backend journal.

Akka persistence is using Cassandra as the journal, as in [`persistence.conf`](https://github.com/cfpb/hmda-platform/blob/master/hmda/src/main/resources/persistence.conf).

```
  persistence

    persistence {
      journal.plugin = "cassandra-journal"

      snapshot-store.plugin = "cassandra-snapshot-store"

      query {
        journal.id = "cassandra-query-journal"
      }
    }
```

- **memo (richard):** currently they just return the status code only, probably we want to give richer info, especially it is waiting for `Future` to be completed after the whole event-processing is done?
- **memo (richard):** Why do PUT and DELETE endpoints not specify `/${LEI}` as the part of URI?
- **memo (richard):** Also, why DELETE needs JSON as the HTTP reqyest body? I think it's unnecessary


####  hmda-public-api `HmdaPublicApi` Actor

- https://github.com/cfpb/hmda-platform/blob/master/docs/v2/api/public-api.md#ts-parsing-and-validation
- https://github.com/cfpb/hmda-platform/blob/master/docs/v2/api/public-api.md#lar-parsing-and-validation
- https://github.com/cfpb/hmda-platform/blob/master/docs/v2/api/public-api.md#hmda-file-parsing-and-validation

 The hmda-public-api HTTP server supports the following endpoints and methods:

- `GET / ` : it just returns a JSON similar to hmda-filing-ap
- `OPTIONS: /` : returns the  "OPTIONS": String` as the HTTP response body, same as hmda-filing-api
- `POST:  /ts/parse`:  accepts JSON represented by case class `TsValidateRequest` //ts as TransmittalSheet?
- `OPTIONS: /ts/parse` : returns the  "OPTIONS": String` as the HTTP response body, same as hmda-filing-api
- `POST: /lar/parse`:  accepts JSON represented by case class LarValidateRequest //ts as TransmittalSheet?
- `OPTIONS: /lar/parse` : returns the  "OPTIONS": String` as the HTTP response body, same as hmda-filing-api
- `POST: /hmda/parse`:  accepts file uploaded as multipart request, using source streaming
- `OPTIONS: /hmda/parse`: returns the  "OPTIONS": String` as the HTTP response body, same as hmda-filing-api
- `POST: /hmda/parse/csv`:  accepts csv file uploaded as multipart request, using source streaming
- `OPTIONS: /hmda/parse`: returns the  "OPTIONS": String` as the HTTP response body, same as hmda-filing-api

- **memo (richard):**  it sems they parse and validate CSV, Lar files but the implementation probably is not completed as `TsValidateRequest` `LarValidateRequest` includes a single String only, not csv

####  hmda-ws-api     - localhost:9080

**memo (richard):** By the way generally they don't need to wrap the result into `ToResponseMarshallable` inside the `complete` directive:

```scala
complete(ToResponseMarshallable(...))
```
but they can do:
```scala
complete(...)
```
because wrapping into `ToResponseMarshallable` is automatically done inside the `complete` directive, as in my article. https://richardimaoka.github.io/blog/akka-http-marshalling-details/

## sbt project = institutional-api

The [institution API](https://github.com/cfpb/hmda-platform/blob/master/docs/v2/api/public-api.md#institutions) page already gives the API.
The route is defined in `InstitutionQueryHttpApi` [(source code)](https://github.com/cfpb/hmda-platform/blob/master/check-digit/src/main/scala/hmda/uli/api/http/ULIHttpApi.scala)

This is not complicated internally, as it currently defines the validation logic in `validation.ULI` but no database or external service are used.

`HmdaInstitutionApi` is the entry point of this project, which starts up an instance of:
  - `InstitutionDBProjection`actor, which is backed up by the following repositories which have DB (Relational DB, as it uses Slick)
    - `InstitutionEmailsRepository`
    - `InstitutionRepository`
    - **memo (richard):**  where is DB connection configured? I don't find it in `hmda-platform/institutions-api/src/main/resources`
 - `HmdaInstitutionQueryApi` actor, and this actor spins up an HTTP server
    - it accesses the repositoris, `InstitutionEmailsRepository` and  `InstitutionRepository` to `findById` and `findByLei` for institions and institution emails

## Kubernetes stuff


