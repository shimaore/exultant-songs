CCNQ4 click-to-dial (via socket.io)
-----------------------------------

To force a call, send a socket.io message `place-call` with the following data:

```
{
  _id: a unique ID for the call, should match ^[\w-]+$
  caller: SIP `username@domain` for the caller
  source: calling number
  destination: target number
  account: account name
  endpoint: endpoint name
}
```
