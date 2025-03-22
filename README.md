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
  - [Allowlisting specific scripts for use with `eval` or `Function`](#allowlisting-specific-scripts-for-use-with-eval-or-function)
- [**Proposed Solution**](#proposed-solution)
  - [Add new CSP directive](#add-new-csp-directive)
  - [Introduce new url-hashes keyword to cover script-src attributes](#introduce-new-url-hashes-keyword-to-cover-script-src-attributes)
  - [Extend script hashes to cover eval](#extend-script-hashes-to-cover-eval)
  - [Add hashes to CSP reporting](#add-hashes-to-csp-reporting)
  - [**Deployment use case examples**](#deployment-use-case-examples)
  - [Single-page applications](#single-page-applications)
  - [Server-side applications](#server-side-applications)
- [**Open questions**](#open-questions)
  - [Should the new script-src-v2 directive override script-src?](#should-the-new-script-src-v2-directive-override-script-src)
- [**Considered alternatives**](#considered-alternatives)
  - [Allowlist external scripts directly by URL, instead of URL hash](#allowlist-external-scripts-directly-by-url-instead-of-url-hash)
  - [Overload the existing unsafe-hashes keyword](#overload-the-existing-unsafe-hashes-keyword)
  - [report-hash keyword](#report-hash-keyword)
- [**Stakeholder Feedback / Opposition**](#stakeholder-feedback--opposition)
- [**References & acknowledgements**](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

We're proposing a new CSP directive to help websites protect themselves against DOM XSS. Developers will be able to allowlist scripts that are allowed to execute through the existing hashes mechanism, that will now extend to cover [script-src](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/script-src) URLs and [eval](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval). This facilitates an easier to deploy, robust CSP policy that mitigates XSS by blocking unallowed inline and eval scripts.

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


### Allowlisting specific scripts for use with `eval` or `Function`

The only existing mechanism to use eval or Function is by enabling them with [unsafe-eval](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/script-src#unsafe_eval_expressions), which allows all scripts. This means that currently any site that needs to use eval must expose itself to eval-based XSS risks. Allowlisting individual scripts would prevent this risk.


## **Proposed Solution**


### Add new CSP directive

Since this proposal adds new ways to allowlist scripts, it would run into backwards compatibility issues in browsers that do not yet support it (e.g. a developer allowlists a URL they’re using in script-src, this will work in browsers that support the new functionality, but will be blocked in older browsers that do not). To address this, we propose adding a new “script-src-v2”\* directive to support the new features. This way sites can add a fallback more permissive script-src (e.g. https: or unsafe-eval) directive that older browsers (which don’t support “script-src-v2” and will ignore it) can fall back to.

\*The naming aims to capture that this is meant to improve coverage of script-src, however this directive does not fully replace script-src, as it doesn’t cover all of what script-src does.

This new directive would support a subset of what script-srcs covers, namely:



*   [nonce](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy#nonce-nonce_value)
*   [<hash_algorithm>-&lt;hash_value>](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy#hash_algorithm-hash_value) (which will now cover eval)
*   [unsafe-hashes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy#unsafe-hashes)
*   [strict-dynamic](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy#strict-dynamic)
*   [unsafe-inline](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy#unsafe-inline)
*   [unsafe-eval](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy#unsafe-eval)
*   The new url-hashes keyword, which will cover script-src URLs, as described in the next section

A point of discussion is whether script-src-v2 replaces script-src if both are set, or we try to enforce both at the same time (see detailed design section below).


### Introduce new url-hashes keyword to cover script-src attributes

Currently, if the CSP header is set, scripts loaded via script-src need to be allowlisted, which can only be done through [nonces](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy#nonce-nonce_value) or SRI. This proposes adding a mechanism to allowlist individual URLs (the initial URL, without following redirects, since following redirects would cause a XSLeak, by exposing whether the URL triggers a redirect) via their hash by adding a new ‘url-hashes’ expression that can be added to script-src.This new url-hashes directive would support both absolute and relative URLs, e.g.:


```
Content-Security-Policy: script-src 'url-hashes' 'sha256-SHA256("https://example.com/script.js")';
<script src="https://example.com/script.js"></script>
```


or 


```
Content-Security-Policy: script-src 'url-hashes' 'sha256-SHA256("script.js")';
<script src="script.js"></script>
```



### Extend script hashes to cover eval

Similarly, scripts run within eval() currently can only be allowed via unsafe-eval, which allows any script, with no mechanism to allowlist only specific ones. This proposes that script hashes should cover scripts loaded via eval, in addition to inline scripts, e.g. given a CSP of `script-src 'sha256-SHA256(foo)'; `permitting `eval(foo);`


### Add hashes to CSP reporting

In order to facilitate easier adoption, script hashes should also be added to CSP reports. This would permit a developer to set a restrictive [report-only CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy-Report-Only), and use the hashes reported to build out a narrowly-defined hash-based allowlist.

This necessitates adding two fields: the hash of the content of a script (for inline/eval scripts), and the hash of the URL (initial URL, without following redirects) of the script (for script src elements).


### **Deployment use case examples**


### Single-page applications

To create a hash-based policy for a static, single-page application, the developer can run tooling to parse the HTML of the application and calculate the hashes of all inline &lt;script> blocks, URLs present in &lt;script src> attributes, and code blocks used in eval or New Function() blocks.

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


## **Open questions**


### Should the new script-src-v2 directive override script-src?

As described in the previous section, this would launch with a new directive so it can be used only on browsers that support the new hashes functionality, while keeping the less strict directives in place to prevent breakage in older browsers that don’t yet support it. This means we have to decide whether the new directive completely replaces script-src in browsers that support it, or whether the browser attempts to enforce both (and prefers the stricter one). The main advantage of completely replacing it is that there is no ambiguity in which directives will be enforced, however it means there is an opportunity for script-src-v2 to be mistakenly configured as less strict than script-src.


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
