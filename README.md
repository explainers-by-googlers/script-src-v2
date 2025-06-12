# Explainer for script-src-v2

This proposal is an early design sketch by the Chrome Secure Web and Network team to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.

## Proponents

- Chrome Secure Web and Network team

## Participate
- https://github.com/explainers-by-googlers/script-src-v2/issues

## Table of Contents

<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [**Goals**](#goals)
- [<strong>Non-goals</strong>](#strongnon-goalsstrong)
- [**Use cases**](#use-cases)
  - [Allowlisting specific URLs for use with script-src](#allowlisting-specific-urls-for-use-with-script-src)
  - [Allowlisting specific scripts for use with `eval` or `eval`-like functions](#allowlisting-specific-scripts-for-use-with-eval-or-eval-like-functions)
- [**Proposed Solution**](#proposed-solution)
  - [Introduce new url-hashes keyword to cover script-src attributes](#introduce-new-url-hashes-keyword-to-cover-script-src-attributes)
  - [Extend script hashes to cover eval and eval-like functions](#extend-script-hashes-to-cover-eval-and-eval-like-functions)
  - [Add hashes to CSP reporting](#add-hashes-to-csp-reporting)
  - [Backwards compatibility](#backwards-compatibility)
    - [Extend script-src with new features, implemented in a way that old browsers can still see a more lax policy](#extend-script-src-with-new-features-implemented-in-a-way-that-old-browsers-can-still-see-a-more-lax-policy)
    - [Implement the new features as a completely new directive](#implement-the-new-features-as-a-completely-new-directive)
  - [**Deployment use case examples**](#deployment-use-case-examples)
  - [Single-page applications](#single-page-applications)
  - [Server-side applications](#server-side-applications)
- [**Considered alternatives**](#considered-alternatives)
  - [Allowlist external scripts directly by URL, instead of URL hash](#allowlist-external-scripts-directly-by-url-instead-of-url-hash)
  - [Overload the existing unsafe-hashes keyword](#overload-the-existing-unsafe-hashes-keyword)
  - [Support only absolute URL hashes](#support-only-absolute-url-hashes)
  - [report-hash keyword](#report-hash-keyword)
- [**Stakeholder Feedback / Opposition**](#stakeholder-feedback--opposition)
- [**References & acknowledgements**](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

We're proposing new CSP features to help websites protect themselves against DOM XSS. Developers will be able to allowlist scripts that are allowed to execute through the existing hashes mechanism, that will now extend to cover [script-src](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/script-src) URLs and [eval](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval) (and other eval-like functions). This facilitates an easier to deploy, robust CSP policy that mitigates XSS by blocking unallowed inline and eval scripts. This new features will be implemented in a backwards compatible way, so sites can set a policy that provides better security in browsers that support them without causing breakage in browsers that do not.

To be secure, a policy needs to permit legitimate scripts to execute, while blocking any scripts that the application doesn't expect. In practice, this means avoiding host-based allowlists and having a strict CSP allowing the execution of scripts by using nonces or hashes.

To be easy-to-deploy, a policy should ideally not require developers to make any changes to their application other than adding CSP (usually via an HTTP header or &lt;meta> tag).


## **Goals**



*   Allow sites to protect themselves from XSS attacks, even if they rely on scripts loaded via eval, or scripts loaded via script-src that change often (i.e. cases where SRI is impractical).
*   Provide a safe alternative for sites that currently use <code>[unsafe-eval](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/script-src#unsafe_eval_expressions)</code>.
*   Implement this in a backwards compatible way, so websites can use it without causing breakage for users of browsers that don’t yet support the feature.


## <strong>Non-goals</strong>



*   Deprecate unsafe-eval (or any other existing CSP directives).


## **Use cases**

XSS is arguably the most dangerous vulnerability class affecting web services for the past 15 years. Content Security Policy was created as a defense-in-depth mechanism to ensure that even when a document suffers from HTML injection, and thus contains arbitrary markup controlled by an attacker, the application will be protected from the execution of untrusted scripts.

The core challenge for CSP is to distinguish between legitimate scripts (intended by the author of the page to execute and usually necessary for an application to function properly) and malicious scripts (potentially injected by an attacker via an HTML injection bug, for example, if a developer has neglected to appropriately escape attacker-controlled data when embedding it in their HTML).


### Allowlisting specific URLs for use with script-src

Sites that want to allowlist specific scripts for use with script-src currently have 2 options, allowlist the specific scripts contents through [subresource integrity](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity), which is not practical for scripts that change often (e.g. analytics scripts), or use [host-source](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy#host-source) to allowlist hostnames, which has the issues [described in further detail below](?tab=t.0#bookmark=id.i59bvq2i29zz). These issues would be addressed if we have a mechanism to allowlist full URLs for script-src.


### Allowlisting specific scripts for use with `eval` or `eval`-like functions

The only existing mechanism to use eval or eval-like functions (Function, and string literals in setTimeout, setInterval, and setImmediate) is by enabling them with [unsafe-eval](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/script-src#unsafe_eval_expressions), which allows all scripts. This means that currently any site that needs to use eval must expose itself to eval-based XSS risks. Allowlisting individual scripts would prevent this risk.


## **Proposed Solution**


### Introduce new url-hashes keyword to cover script-src attributes

Currently, if the CSP header is set, scripts loaded via script-src need to be allowlisted, which can only be done through [nonces](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy#nonce-nonce_value), SRI, or hostname sourcelists. This proposes adding a mechanism to allowlist individual URLs (the initial URL, without following redirects, since following redirects would cause a XSLeak, by exposing whether the URL triggers a redirect) via their hash by including their hash (prefixed with url-<hashalgorithm>) in script-src. URL hashes would support both absolute and relative URLs, e.g.:


```
Content-Security-Policy: script-src 'url-sha256-SHA256("https://example.com/script.js")';
<script src="https://example.com/script.js"></script>
```


or 


```
Content-Security-Policy: script-src 'url-sha256-SHA256("script.js")';
<script src="script.js"></script>
```



### Extend script hashes to cover eval and eval-like functions

Similarly, scripts run within eval() currently can only be allowed via unsafe-eval, which allows any script, with no mechanism to allowlist only specific ones. This proposes that script hashes should cover scripts loaded via eval (Function, or string literals in setTimeout, setInterval, and setImmediate), in addition to inline scripts, e.g. given a CSP of `script-src 'eval-sha256-SHA256(foo)';` permitting `eval(foo);`


Prefixes in hashes are necessary to distinguish these from existing hashes supported in script-src. This is useful to prevent scripts from also being unintentionally allowlisted as inline scripts, and is also required to implement backwards compatibility (so browsers that understand the new features can detect when they are being used).


### Add hashes to CSP reporting

In order to facilitate easier adoption, script hashes should also be added to CSP reports. This would permit a developer to set a restrictive [report-only CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy-Report-Only), and use the hashes reported to build out a narrowly-defined hash-based allowlist.

This necessitates adding two fields: the hash of the content of a script (for inline/eval scripts), and the hash of the URL (initial URL, without following redirects) of the script (for script src elements).


### Backwards compatibility
One of the goals is for the new features to be deployable on sites without breaking on older browsers. There are two ways to achieve this:


#### Extend script-src with new features, implemented in a way that old browsers can still see a more lax policy
This would extend the existing script-src directive with 2 new keywords, `eval-<hashalgorithm>-<hash>` and `url-<hashalgorithm>-<hash>`. For eval, backwards compatibility would be achieved by browsers that support the new features ignore `unsafe-eval` if one or more eval hashes are present (which matches the existing behavior for `unsafe-inline`, which is [ignored if script hashes are present](https://www.w3.org/TR/CSP3/#allow-all-inline)). This would allow a site that needs to use eval to set both unsafe-eval and the script hashes. Browsers that don't support the new feature would still work due to the presence of unsafe-eval (and ignore the hashes since they don't understand that keyword), while browsers that support hashes would ignore `unsafe-eval`. In practice this would look like:


```
Content-Security-Policy: script-src 'eval-sha256-SHA256("console.log('Hello World')';
<script>
eval("console.log('Hello World')");
</script>
```


For URL hashes, a similar approach can be followed. Browsers that support the new features can ignore host based allowlisiting if a URL hash is present. Unlike eval though, this leaves some open questions about interactions with other directives, notably `strict-dynamic`. A common fallback for sites that use URL hashes would be to include `https:` in the fallback policy. However sites might require `strict-dynamic` in order to use URL hashes, and they can't set a policy like `script-src https: strict-dynamic, url-sha256-<hash>` since `strict-dynamic` would cause `https:` to be [ignored](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/script-src#strict-dynamic). This can be solved by either:

- Adding a new keyword that enables strict-dynamic like behavior for URL hashes, but is ignored in older browsers, e.g. `strict-dynamic-hashes`.
  -e.g.: For a site that needs `https: 'unsafe-inline'` in older browsers and `'strict-dynamic-hashes' 'url-sha256(example.com/script.js)'` in browsers that support hashes, it can set `https: 'unsafe-inline' 'strict-dynamic-hashes' 'url-sha256(example.com/script.js)'` as its policy. Older browsers will ignore `strict-dynamic-hashes` and the url hash, while newer browsers will ignore `https:` due to the presence of a hash and `unsafe-inline` due to the presence of `strict-dynamic-hashes`.
- Having the presence of URL hashes imply strict-dynamic, so the keyword is not necessary. This would be the easiest behavior to explain, but it means that sites using hashes lose the option to disable `strict-dynamic` behavior, so sites that don't need it, but want to use URL hashes have a policy that is more lax than they really need.
  -If implemented this way, strict-dynamic behavior could also be limited to scripts loaded via a hashed URL.
- Having the presence of URL hashes imply strict-dynamic, and providing an opt-out keyword, e.g. `no-strict-dynamic-hashes`.

#### Implement the new features as a completely new directive
The new features can be implemented as part of a separate directive (e.g. `script-src-v2`) that completely replaces `script-src` if it's set. This has the advantage of making the policy easier to read since it's immediately obvious what applies in browsers that are compatible with the new features, and what is the fallback for browsers that are not. On the other hand, the main drawback to this approach is that moves the complexity to the [fallback algorithm](https://www.w3.org/TR/CSP3/#directive-fallback-list).

Both `script-src-elem` and `script-src-attr` fallback directly to `script-src` so they can be handled by either introducing a replacement that supports the new features and replaces the other one if set(e.g. `script-src-elem-v2`), or by completely ignoring them if `script-src-v2` is set.

`worker-src`, `child-src`, and `default-src` interact with `script-src`, but also with other directives, so adding a "v2" version of them leads to an overly complicated algorithm. Alternatively we can add support for url hashes directly on these directives, but that means the only way to have a backup policy for browsers that don't support them, is to not rely on the fallback algorithm and use `script-src` directly. Finally, we can also choose to not support the new features in these, but this leads to a scenario where sites that want to use URL hashes for workers have to *not* set `worker-src` and are forced to rely on the fallback to `script-src`

### **Deployment use case examples**


### Single-page applications

To create a hash-based policy for a static, single-page application, the developer can run tooling to parse the HTML of the application and calculate the hashes of all inline &lt;script> blocks, URLs present in &lt;script src> attributes, and code blocks used in eval, New Function(), or string literals in setTimeout, setInterval, and setImmediate blocks.

The tooling can generate a list of hashes and potentially automatically insert an HTML &lt;meta> tag with their values, e.g. &lt;meta http-equiv="Content-Security-Policy" content="script-src ‘sha256-abc...” url-hashes 'sha256-xyz…'">. The developer can optionally add the 'strict-dynamic' keyword to permit allowlisted scripts to transitively load additional scripts at runtime.

This could be achieved at compilation time. Any time the application is redeployed, if the scripts have changed, the hashes will be updated to match the new script values.


### Server-side applications

To semi-automatically create a strong hash-based policy for an application with a server-side component developers can use the following approach:



*   Set an application-wide report-only CSP of script-src 'report-hashes' 'report-sample'; report-uri /csp-reports in a production environment.
*   Collect the reported hashes and create a list of all the hashes of scripts executing in the production environment.
*   The developer could optionally investigate the reported hashes (to verify that they correspond to expected application markup) before adding them to the CSP directive and removing report-only. URLs could also be included in the report to facilitate this.
*   Create an enforcing policy listing all the collected hashes.

This process could also be delegated to a third-party service - the developer would only need to set a reporting CSP header and after a few days/weeks would receive a list of hashes to include in their enforcing CSP.

After the application enforces a hash-based CSP, if a developer adds a new script or modifies an existing one, they will immediately notice that the script is blocked from executing and needs a new hash to be added to the policy to be enabled.

This carries a significant promise of allowing the deployment of CSP in legacy applications which don't undergo frequent changes, but which might otherwise process sensitive data.


## **Considered alternatives**


### Allowlist external scripts directly by URL, instead of URL hash

The `host-sources` directive allows allowlisting by hostname, but has the following limitations:



*   host-sources does not include anything after the path (e.g. can allowlist /foo/bar.js, but not /jsonp?callback=foo)
*   host-sources follows redirects (i.e. the redirect target needs to also be part of the allowlist), which we do not want for the new directive
*   ‘strict-dynamic’ makes the browser ignore host-sources.

While it would be possible to introduce a new URL-based directive with different semantics (e.g. that includes URL parameters), including a large list of URLs may exceed the response header limit quickly. Some cursory investigation of HTTP Archive data suggests that, for large sites, hashes result in significantly shorter allowlists than do raw URLs.


### Overload the existing unsafe-hashes keyword

[unsafe-hashes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy#unsafe-hashes) is an existing directive that allows hashes to apply to event handlers (if not set, hashes currently only apply to inline &lt;script> blocks). It was introduced [[1](https://github.com/w3c/webappsec-csp/issues/13#issuecomment-186708533), [2](https://docs.google.com/document/d/1_nYS4gWYO2Oh8rYDyPglXIKNsgCRVhmjHqWlTAHst7c/edit?tab=t.0#heading=h.h95n37p306j5)] because allowlisting inline scripts for event handlers is considered unsafe even when using hashes, as a script that can be safe when used for a particular event handler (e.g. `<button onclick="transferAllMyMoney()">Transfer all my money&lt;/button>`) might not be safe for a different handler (e.g. `<image src="doesnotexist.test" onerror="transferAllMyMoney() />`) or as an inline script.

For simplicity, we could reuse `unsafe-hashes` to have hashes apply to eval and script URLs, however in the eval case, this would mean that scripts would be implicitly allowlisted for event handlers, which is not intended. For this reason we decided not to require unsafe-hashes for eval. This design implicitly allows scripts allowlisted for eval via hashes to also be used in an inline block. This seems fine, but if it’s determined that we do need a keyword to split allowlisting scripts for eval from allowlisting them for inline use, we can add a separate eval-hashes keyword.

We propose using the new url-hashes keyword instead of reusing unsafe-hashes to avoid confusion from using the same directive to allow two different uses (and since allowlisting urls is not, in fact, “unsafe”).


### report-hash keyword

There is also [a proposal](https://github.com/w3c/webappsec-csp/pull/693) for reporting hashes of content of all scripts, including script src elements. That proposal aims to audit the content of all scripts running on the page, while the reporting in this proposal aims to facilitate the deployment of this proposal's policy. Reporting also happens at different times in the two proposals (request time or parsing/eval time for this CSP-building proposal, and response time for the other proposal).


## **Stakeholder Feedback / Opposition**

No signals yet


## **References & acknowledgements**

Many thanks for valuable feedback and advice from:



*   Arthur Janc
*   David Dworken
*   Domenic Denicola
*   Lukas Weichselbaum
*   Mike West
