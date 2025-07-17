# **Incentivizing `MessageEvent` origin checks**

**TL;DR**: When processing a message delivered via `Window.postMessage()`, we should encourage developers to validate the message source by preventing access to `MessageEvent.data` until `MessageEvent.origin` is accessed.

## **A Problem**

`Window.postMessage()` is a fairly critical piece of web infrastructure, creating a communication channel between cross-origin windows that would otherwise remain fairly isolated from each other. This enables composition and collaboration in ways that directly support the needs of web developers, forming the foundation of user-facing multi-party flows like login and payments.

Unfortunately, the API is poorly-designed from a security perspective. The sender is required to specify the origin to which a message ought to be delivered, but the recipient has no such obligation despite the clear risks of accepting messages from arbitrary senders. The `MessageEvent` they receive does have an `origin` attribute, but it's unfortunately quite common for developers to simply grab the `data` attribute and run off without validating its provenance.

This isn't a new problem: among other papers Hanna, et al's "[The Emperor’s New APIs: On the(In)Secure Usage of New Client-side Primitives](https://webblaze.cs.berkeley.edu/papers/w2sp2010ena.pdf) identified it in 2010, Son and Shmatikov's"[The Postman Always Rings Twice: Attacking and Defending postMessage in HTML5 Websites](https://www.cs.utexas.edu/~shmat/shmat_ndss13postman.pdf)" dug in a bit more, and Steffens and Stock's "[PMForce: Systematically Analyzing `postMessage` Handlers at Scale](https://swag.cispa.saarland/papers/steffens2020pmforce.pdf) confirmed it remains an issue on the modern web.

It would be ideal to change the API's shape somewhat dramatically in order to ensure that recipients are required to make decisions about the set of origins from which they'll accept messages at the same time they register an event handler. A declarative message-reception API relying on some [URLPattern](https://developer.mozilla.org/en-US/docs/Web/API/URL_Pattern_API)-like origin matcher (or even something simpler\!) would be a very reasonable thing to introduce. It would, however, require developers to change their working code. Perhaps we can help motivate refactoring, either towards an alternative or simply towards best practice with the existing API.

## **A Proposal**

When handling messages delivered via `Window.postMessage()`, we should force developers to do some kind of check against the `MessageEvent.origin` attribute before allowing them to access the contents of `MessageEvent.data`. We can't exactly ensure that developers perform a robust check that meets the requirements of their use case, but we can at least require them to *look* at the `origin` attribute by hiding the `data` attribute's value until they do. That is:

```javascript
// Oh noes!
window.addEventListener('message', e => {
  const incomingData = e.data; // This would return `null`, as `origin`
                               // hasn't yet been accessed.
  ...
});

// Better!
window.addEventListener('message', e => {
  if (!isReasonableOrigin(e.origin)) {
    return false;
  }
  const incomingData = e.data; // This would return data, as `origin`
                               // was accessed prior to its execution.
  ...
});
```

This would require three small changes to `MessageEvent`, and one small change to the [window post message steps](https://html.spec.whatwg.org/multipage/web-messaging.html#window-post-message-steps):

1. `MessageEvent` will hold an internal slot, `[[Origin Check Required]]`, which is `false` unless otherwise specified.

2. The `MessageEvent.data` attribute's getter will become slightly more complicated, a la:  
   1. If `[[Origin Check Required]]` is `true`, return `null`.  
   2. Otherwise, return the value of the `data` attribute.

3. The `MessageEvent.origin` attribute's getter will likewise become slightly more complicated:  
   1. Set `[[Origin Check Required]]` to `false`.  
   2. Return the value of the `origin` attribute.

4. We'll change step 8.7 of the [window post message steps](https://html.spec.whatwg.org/multipage/web-messaging.html#window-post-message-steps) to include the following addition:  
   1. [Fire an event](https://dom.spec.whatwg.org/#concept-event-fire) named [message](https://html.spec.whatwg.org/multipage/indices.html#event-message) at `targetWindow`, using [MessageEvent](https://html.spec.whatwg.org/multipage/comms.html#messageevent), with the [origin](https://html.spec.whatwg.org/multipage/comms.html#dom-messageevent-origin) attribute initialized to `origin`, the [source](https://html.spec.whatwg.org/multipage/comms.html#dom-messageevent-source) attribute initialized to `source`, the [data](https://html.spec.whatwg.org/multipage/comms.html#dom-messageevent-data) attribute initialized to `messageClone`<ins>**, the `[[Origin Check Required]]` slot initialized to `true`**</ins>, and the [ports](https://html.spec.whatwg.org/multipage/comms.html#dom-messageevent-ports) attribute initialized to `newPorts`.

That's it. This will force developers to touch the `origin` attribute before doing something useful with the `data` attribute. It's a speedbump, but with documentation and good devtools messaging, it's one that might improve the quality of existing API usage.

## **FAQ**

### **Surely this would break pages.**

That's not a question, but yes. It would. That means it's a deprecation we'd need to ramp up to, with some stepping stones along the way involving devtools messages, documentation updates, etc. We could likely also find non-destructive ways of creating incentives for developers: for example, we could simply slow down the `data` getter if `origin` hasn't been accessed. \~500ms delays would get developers' attention without overly burdening users.

We can also take steps to reduce the scope of impact. Perhaps we can skip the check if the event is same-origin with the receiving document? Same-site?

### **What's a reasonable pattern we should encourage developers to use?**

In the best case, developers can allowlist certain origins, just as they do for `postMessage()` today. The `URLPattern` API can be helpful here:

```javascript
function isReasonableOrigin(origin) {
  try {
    const url = new URL(origin);
    const validOrigins = new URLPattern({
      protocol: "https",
      hostname: "*.totally-trustworthy.example"
    });
    return validOrigins.test(url);
  } catch(e) {
    return false; // Or `true`, I suppose, if your application needs to
                  // work with opaque origins.
  }
}

window.addEventListener('message', e => {
  if (isReasonableOrigin(e.origin)) {
    // Exciting actions go here...
  }
});
```
