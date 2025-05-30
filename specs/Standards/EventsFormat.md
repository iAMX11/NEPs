# Events Format

## [NEP-297](https://github.com/near/NEPs/blob/master/neps/nep-0297.md)

Version `1.0.0`

## Summary

Events format is a standard interface for tracking contract activity.
This document is a meta-part of other standards, such as [NEP-141](https://github.com/near/NEPs/issues/141) or [NEP-171](https://github.com/near/NEPs/discussions/171).

## Motivation

Apps usually perform many similar actions.
Each app may have its own way of performing these actions, introducing inconsistency in capturing these events.

NEAR and third-party applications need to track these and similar events consistently.
If not, tracking state across many apps becomes infeasible.
Events address this issue, providing other applications with the needed standardized data.

Initial discussion is [here](https://github.com/near/NEPs/issues/254).

## Events

Many apps use different interfaces that represent the same action.
This interface standardizes that process by introducing event logs.

Events use the standard logs capability of NEAR.
Events are log entries that start with the `EVENT_JSON:` prefix followed by a single valid JSON string.  
JSON string may have any number of space characters in the beginning, the middle, or the end of the string.
It's guaranteed that space characters do not break its parsing.
All the examples below are pretty-formatted for better readability.

JSON string should have the following interface:

```ts
// Interface to capture data about an event
// Arguments
// * `standard`: name of standard, e.g. nep171
// * `version`: e.g. 1.0.0
// * `event`: type of the event, e.g. nft_mint
// * `data`: associate event data. Strictly typed for each set {standard, version, event} inside corresponding NEP
interface EventLogData {
    standard: string,
    version: string,
    event: string,
    data?: unknown,
}
```

Thus, to emit an event, you only need to log a string following the rules above. Here is a barebones example using Rust SDK `near_sdk::log!` macro (security note: prefer using `serde_json` or alternatives to serialize the JSON string to avoid potential injections and corrupted events):

```rust
use near_sdk::log;

// ...
log!(
    r#"EVENT_JSON:{"standard": "nepXXX", "version": "1.0.0", "event": "YYY", "data": {"token_id": "{}"}}"#,
    token_id
);
// ...
```

#### Valid event logs

```js
EVENT_JSON:{
    "standard": "nepXXX",
    "version": "1.0.0",
    "event": "xyz_is_triggered"
}
```

```js
EVENT_JSON:{
    "standard": "nepXXX",
    "version": "1.0.0",
    "event": "xyz_is_triggered",
    "data": {
        "triggered_by": "foundation.near"
    }
}
```

#### Invalid event logs

* Two events in a single log entry (instead, call `log` for each individual event)

```js
EVENT_JSON:{
    "standard": "nepXXX",
    "version": "1.0.0",
    "event": "abc_is_triggered"
}
EVENT_JSON:{
    "standard": "nepXXX",
    "version": "1.0.0",
    "event": "xyz_is_triggered"
}
```

* Invalid JSON data

```js
EVENT_JSON:invalid json
```

* Missing required fields `standard`, `version` or `event`

```js
EVENT_JSON:{
    "standard": "nepXXX",
    "event": "xyz_is_triggered",
    "data": {
        "triggered_by": "foundation.near"
    }
}
```

## Drawbacks

There is a known limitation of 16kb strings when capturing logs.
This impacts the amount of events that can be processed.
