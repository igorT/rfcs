- Start Date: (fill me in with today's date, 2019-03-09)
- Relevant Team(s): data
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)
    
# Request State Service
    

## Summary

Currently, Ember Data internally uses a complicated state machine abstraction to help records keep track of what state they are in and expose state flags. A lot of those flags try to expose request state. Instead of relying on a complicated inflexible state machine, this RFC proposes exposing a Request State service which would expose inflight and completed requests.

# Motivation

Currently DS.Model has several flags that expose information about either inflight requests or completed requests. `isReloading` and `isSaving` expose inflight request state, while `isLoaded`, `isValid`, `isError`, expose state about the last completed request. While these flags are very convenient, the have several drawbacks:

- They rely on Ember Data's internal state machine which is not app controllable and is very inflexible
- They assume a single source, and as we try to support more advanced use cases will prove insufficient. For example, if you are saving records both in IndexDB and on the backend, `isSaving` as a flag is insufficient.

This RFC proposes creating a RequestState service which would allow us to rewrite the current flags as helper methods and move us away from the internal state machine implementations. It would also allow users to handle cases not covered by the rigid model flags, and give us a place to experiment with a multi source approach. Moreover, such a service would also allow for a better debugging experience and could be easily exposed in the Ember Inspector, allowing for a much nicer debugging experience compared to the existing Promises tab.

## Detailed design

This RFC exposes the minimal necessary set of APIs needed to reimplement the existing flags on DS.Models. In particular, it only covers requests pertaining to `findRecord` , `reload`

and `save` methods. Those are also easier to design and implement because they match up to a single record and don't require a more complex `Query` interface. We would expect RequestState Services scope of responsibility to grow in the future to where it would handle all of requests.

Design of the RequestState service:

```ts
// no id yet
interface Request {
  data: Operation[];
  options: any
}

interface QueryRequest extends Request {
  data: QueryOperation;
}

interface MutationRequest extends Request {
  data: MutationOperation[];
}

interface Operation {
  op: string
}

interface Query extends Operation {};

interface FindRecordQuery extends Query {
  op: 'findRecord'
  identifier: RecordIdentifier
  options: any
}

interface Mutation extends Operation {};

interface SaveRecordMutation extends Mutation {
  op: 'saveRecord'
  identifier: RecordIdentifier
  options: any
}

interface RequestState {
  state: 'pending' | 'fulfilled' | 'rejected'
  type: 'query' | 'mutation'
  request: Request;
  response?: Response;
}

interface Response {
  rawData: unknown;
  data: unknown;
}

function unsubscribe(token: UnsubscribeToken): void

class RequestStateService {
  getPendingRequests(recordIdentifier: RecordIdentifier): Request[]
  getLastRequest(recordIdentifier: RecordIdentifier): Request | null
  subscribe(recordIdentifier: RecordIdentifier, callback: Function): UnsubscribeToken
}
```

The subscription methods are needed in order the model to learn about changes to network states. Once tracked properties exposes a public way for manipulating Tags, we will likely be able to move away from the subscription mechanism to a nicer notification mechanism.

Expose a service on the store

```ts
class Store {
  getRequestStateService(): RequestStateService
}   
```

Using these  we can reimplement the current `isSaving` method on `DS.Model`

The subscription mechanism is deliberately somewhat klunky in anticipation of replacing it with an automatic tracked properties solution.

```ts
DS.Model.extend({

  init() {
    this._requestSubscriptions = {};
  } 

  isSaving: computed(function () {
    this._subscribeRequests('isSaving');
    let requests = this.store.getRequestStateService().getPending(identifierForModel(this));
    return !!requests.find((req) => req.data.op === 'saveRecord');
  }),

  _subscribeRequests(key) {
    if (!this._requestSubscriptions[key]) {
      this._requestSubscriptions[key] = true;
      this.store.getRequestStateService().subscribe(identifierForModel(this), () => {
        this.notifyPropertyChange(key);
      })
    }
  }
});
```

## How we teach this

Initially we would write up the API docs for the request. Once several followup RFCs which made the design ergonomic and added other use cases,
we would write up a guide entry showing common ways to handle request based use cases with the request state service.

## Drawbacks

We are exposing more internal state as public api.

The subscription mechanism is not extremely ergonomic.

Exposing the raw results in request objects might lead to apps using the raw payloads and punching wholes through our existing api boundaries
