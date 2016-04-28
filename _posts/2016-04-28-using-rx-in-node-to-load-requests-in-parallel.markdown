---
published: true
title: Using Rx in Node to load requests in parallel
layout: post
---
# Using Rx in Node to load requests in parallel

Sometimes you need to load some requests in parallel before returning the joint results. 
With RxJs, this is pretty simple to do.

As a skeleton, we use the next code:


```javascript
communityContract.getClaimsCount((error, claimsCount) => {
    if (error) {
        rxHelper.handleResponse(error, null, observer);
    } else {
        let claims = [];
        Rx.Observable
            .range(1, claimsCount)
            .flatMap((counter) => {
                // todo
            })
            .subscribe((combinedResult) => {
                // todo
            },
            (error) => {
                rxHelper.handleResponse(error, null, observer);
            },
            () => {
                rxHelper.handleResponse(error, claims, observer);
            });
        }
    });
},
(error) => {
    rxHelper.handleResponse(error, null, observer);
});
```

First, we call the `communityContract.getClaimsCount` to get the number of claims to retrieve. 
In this specific example, the backend is a BlockChain which can not return dynamic structures. 
The `communityContract` is a class to call a smart contract on the BlockChain. 
It follows the Node convention invoking the callback when the results are retrieved.

We then create an observable stream using the `range` operator. For each value from 1 up until claimsCount one observer.next() will be called.
We are then going to use the `flatMap` operator to load all claim properties for 1 claim in parallel. 

(By the way, the `rxHelper` is just a helper class which implements some convenient methods. The full code is included at the end of this blog).

The `flatMap` operator flattens a series of child observables to the main observable stream. In it, we can call the necessary requests to get all properties from a claim:
```javascript
.flatMap((counter) => {
    let index = counter - 1; // zero based
    return Rx.Observable.just(index)
        .forkJoin(
            Rx.Observable.fromNodeCallback(communityContract.getClaim)(index),
            Rx.Observable.fromNodeCallback(communityContract.getClaimIPFS)(index),
            Rx.Observable.fromNodeCallback(communityContract.getClaimStatus)(index)
        );
})
```
RxJs gives us the handy `Rx.Observable.fromNodeCallback` shortcut to transform a method with a Node callback to an observable.

The next step in our flow is subscribing to the observable stream in order to get the combined results:
 ```javascript
.subscribe((combinedResult) => {
    let claim = mapGetClaimResultToClaim(combinedResult[1]);

    let claimIPFS = mapGetClaimIPFSResultToClaimIPFS(combinedResult[2]);
    claim.imageHashIPFS = claimIPFS.imageHashIPFS;
    claim.imageTypeIPFS = claimIPFS.imageTypeIPFS;

    let claimStatus = mapGetClaimStatusResultToClaimStatus(combinedResult[3]);
    claim.claimStatus = claimStatus.claimStatus;
    claim.percentageAccepted = claimStatus.percentageAccepted;
    claim.percentageVoted = claimStatus.percentageVoted;

    claims.push(claim);
},
```
The `combinedResult` is an array with the results for all requests which were performed in the `flatMap` step.
Element `0` contains the `index`, element `1` the results for the `communityContract.getClaim` request etc.
In the `subscribe` the results from all requests are combined to the `claim` which is then pushed to the `claims` array which will be returned to the caller.

And that's it. We now have working code to first get the number of claims, and then a flow to get each claim by invoking 3 extra requests. 
Rx handles the async orchestration.

### This is the full code: 
```javascript
communityContract.getClaimsCount((error, claimsCount) => {
    if (error) {
        rxHelper.handleResponse(error, null, observer);
    } else {
        let claims = [];
        Rx.Observable
            .range(1, claimsCount)
            .flatMap((counter) => {
                let index = counter - 1; // zero based
                return Rx.Observable.just(claimId)
                    .forkJoin(
                        Rx.Observable.fromNodeCallback(communityContract.getClaim)(claimId),
                        Rx.Observable.fromNodeCallback(communityContract.getClaimIPFS)(claimId),
                        Rx.Observable.fromNodeCallback(communityContract.getClaimStatus)(claimId)
                    );
            })
            .subscribe((combinedResult) => {
                let claim = mapGetClaimResultToClaim(combinedResult[1]);

                let claimIPFS = mapGetClaimIPFSResultToClaimIPFS(combinedResult[2]);
                claim.imageHashIPFS = claimIPFS.imageHashIPFS;
                claim.imageTypeIPFS = claimIPFS.imageTypeIPFS;

                let claimStatus = mapGetClaimStatusResultToClaimStatus(combinedResult[3]);
                claim.claimStatus = claimStatus.claimStatus;
                claim.percentageAccepted = claimStatus.percentageAccepted;
                claim.percentageVoted = claimStatus.percentageVoted;

                claims.push(claim);
            },
            (error) => {
                rxHelper.handleResponse(error, null, observer);
            },
            () => {
                claims = _.sortBy(claims, (claim) => {
                    return claim.timestamp;
                }).reverse();
                rxHelper.handleResponse(error, claims, observer);
            });
        }
    });
},
(error) => {
    rxHelper.handleResponse(error, null, observer);
});

```

### And this is the full code of the `rxHelper`:
```javascript
'use strict';

module.exports = {
    handleResponse: handleResponse,
}

let logger = require('../setup/logger');

function handleResponse(error, response, observer) {
    if (error) {
        logger.error(error);
        observer.onError(error);
    } else {
        observer.onNext(response);
        observer.onCompleted();
    }
}
```