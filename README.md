# [See Other]! 😑 → 🤖

A [service worker] for *very cachable web3 websites over web2*

> You want to host your website on IPFS, and your want it to work nicely over https. You set up a dnslink record and point your domain at an IPFS gateway and it all works! RAD! 
>
> but look at those cache headers 😑 ...they could be `permenant` but you playin' (with mutable urls) 

If you ask an IPFS gateway to translate a mutable url to the permenant dnslink/cid one, you can't cache the response! *What if a service worker transparently redirected all requests to the content addressed version behind the scenes!*

The see-other service worker looks up the [dnslink] for your site and redirects request for `https://mysite.com/big.jpg` to the cache friendly `https://ipfs.io/ipfs/<cid>/big.jpg` url. Much like the gateway would do, but by doing it on the client you can cache the heck out of those responses 👨‍🍳👌✨

With a minor abuse of the [Cache] api, it handles periodically refreshing dnslink too.

If you are totally lost, then https://blog.fleek.co/posts/immutable-ipfs is a good primer on permenant vs mutable URs

## Getting started

**Warning: ⚠️ a demo of an idea - you should really all be using [ipfs-companion] ⚠️**

- Copy `see-other.js` into the root of your static website project. Edit as you see fit.
- Register the service worker from a script tag or your main js bundle.
```
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('see-other.js');
}
```
- Set up a build process that adds your site to IPFS and updates the [dnslink]. We use https://github.com/ipfs-shipyard/ipfs-github-action and dnsimple. Or see [Fleek](https://fleek.co/)

See the [example](example) directory. You can run `npm start` in this project to serve it up.

## Why?

- 🤓 Human friendly document url preserved (e.g. https://blog.ipfs.io).
- 🤖 Cache friendly CID versioned urls for all subresources with no change to your build process. It's a drop-in improvement.
- 🧙‍♀️ No magic. Just aligning good old http caching with the permenant web. Your browser will incrementally permenantly cache the resources you fetch. When you update the dnslink, the service worker will update the CID to fetch the subresources from.
- 🔥 Will make IPFS hosted webites feel fast for folks not yet on the web3 train.
- 🤝 Less busy work for the gateways. No request is much less work than a 304, when you have to ipfs.get to figure out the answer.


## How?

The very first page load will fetch and install the service worker. It will look up the dnslink for your domain, and cache the repsonse.

When you navigate to the next page, the service worker will be in control and all image/css/js requests will be redirected to `https://ipfs.io/<current cid from dnslink>/path/to/thing`, which come with the `permenant` cache header.

By the third page view, you are now using the permenanently cached local versions of all resources you've seen before and completly skipping the network requests. 👨‍🍳👌✨


## What the hack?

The current dnslink for the site is fetched via `https://ipfs.io/api/v0/dns?arg=<current domain>` when the service worker starts up. Service workers are stateless and isolated from the DOM, so we can't store the current dnslink in a global variable, nor in localstorage. We do have access to the [Cache] api, so we save a copy of the dnslink response there, so we dont have to fetch if for every request.

But with great caching comes cache invalidation. A simple path is to just re-check the dnslink periodically. The Cache api preserves the heeders of the request, so in theory we could grab the `date` header from the response and see how old it is. But CORS strikes again! As the dnslink api request is cross-origin, we dont get to see the headers. It's an [opaque response].

However, we can copy the Reponse and just new up our very own headers for it, as we just want know how long this entry has been in the cache, so we add a `x-cached-at` header, and check that on every request to determine when dnslink is too old and we should check again.


## References

- js-ipfs in a service-worker - https://github.com/ipfs/js-ipfs/tree/master/examples/browser-service-worker
- intro to service-worker - https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers
- the service-worker lifecycle - https://developers.google.com/web/fundamentals/primers/service-workers/lifecycle
- Example that updates request urls - https://lea.verou.me/2017/10/different-remote-and-local-resource-urls-with-service-workers/
- Perf tips - https://developers.google.com/web/fundamentals/primers/service-workers/high-performance-loading
- Setting custom headers - https://stackoverflow.com/questions/42585254/is-it-possible-to-modify-service-worker-cache-response-headers


[See Other]: https://en.wikipedia.org/wiki/HTTP_303
[service worker]: https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers
[Cache]: https://developer.mozilla.org/en-US/docs/Web/API/Cache
[ipfs-companion]: https://github.com/ipfs-shipyard/ipfs-companion
[dnslink]: https://docs.ipfs.io/concepts/dnslink/#publish-using-a-subdomain