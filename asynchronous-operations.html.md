---
title: Asynchronous Operations
---

## <a id='blocking'></a>Blocking Operations ##

### Provisioning ###

While a synchronous provision is in progress:

- Requests to create instances with duplicate names are not prevented until creation of one has succeeded. Once the first is saved to Cloud Controller database, subsequent success responses will fail and orphan mitigation initiated. To these requests Cloud Controller returns 400 "The service instance name is taken: myasync". CLI interprets these responses as a success for idempotent scriptability, with message "Service myasync already exists".
- Requests to update, bind, unbind, and delete fail cannot be made as the service instance has not be saved to the database. For delete only, CLI returns a success for idempotent scriptability, with message "Service myasync does not exist."

While an asynchronous provision is in progress:

- Requests to create instances with duplicate names are not prevented until creation of one has succeeded. Once a 202 response is received for a provision request the instance is created in Cloud Controller database. Though the instance may not be created or usable, requests to create duplicate instances will fail with response 400 "The service instance name is taken: mydb". CLI interprets these responses as a success for idempotent scriptability, with message "Service myasync already exists".
- Requests to update plan fail with response 400 "Another operation for this service instance is in progress."
- Requests to bind fail with response 400 "Another operation for this service instance is in progress."
- Requests to unbind can not be made because there is not binding resource to delete. The CLI returns a success for idempotent scriptability, with message "Binding between myasync and async-broker did not exist".
- Requests to delete fail with response 400 "Another operation for this service instance is in progress."

## <a id='blocking'></a>Asynchronous Operations ##

### Polling Interval ###
When making an asynchronous request (request in which the client specifies accepts_incomplete=true), the broker can return an optional value for expected polling interval. The polling interval is set by default to 60 seconds. This value will be used for polling the opration state, but can be overwitten in the following cases:

- If the polling interval value provided is lower the minimal value which default is 60 seconds. This could be defined by the cloud controller operator.
- If the polling interval value provided exeeds the maximal value of 86400 seconds (24 hours)

### Polling Timeout ###
When making an asynchronous request (request in which the client specifies accepts_incomplete=true), the client will be able to make a maximal number of polling attempts before declaring the operation as failed. This number can be configured by the cloud controller operator, and is set by default to 25.
