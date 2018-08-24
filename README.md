# Safari Intelligent Tracking Prevention

<!-- TOC depthFrom:2 -->

- [Statistics collection](#statistics-collection)
  - [Subframes and redirects](#subframes-and-redirects)
  - [Hardcoded SSO exceptions](#hardcoded-sso-exceptions)
    - [Dow Jones](#dow-jones)
    - [Adobe](#adobe)
- [Classification](#classification)
  - [Prevalence](#prevalence)
    - [Vector Threshold](#vector-threshold)
    - [Core Prediction](#core-prediction)
  - [User interaction](#user-interaction)
  - [Bounce trackers and redirect collusion](#bounce-trackers-and-redirect-collusion)
- [Effects of classification](#effects-of-classification)
  - [Access to cookies in a third-party context](#access-to-cookies-in-a-third-party-context)
  - [Automatic cookie access for popups](#automatic-cookie-access-for-popups)
  - [Purging cookies](#purging-cookies)
  - [First-party cookies and redirects](#first-party-cookies-and-redirects)
  - [Stripping Referer Header](#stripping-referer-header)
- [Storage access](#storage-access)
  - [hasStorageAccess](#hasstorageaccess)
  - [requestStorageAccess](#requeststorageaccess)
  - [Lifetime of storage access](#lifetime-of-storage-access)
  - [Exceptions](#exceptions)
    - [Popups after requestStorageAccess](#popups-after-requeststorageaccess)
- [Miscellaneous](#miscellaneous)
  - [Resetting ITP statistics and classification](#resetting-itp-statistics-and-classification)
  - [Pruning Statistics](#pruning-statistics)
  - [ITP Debug mode](#itp-debug-mode)
  - [Telemetry](#telemetry)

<!-- /TOC -->


## Statistics collection

### Subframes and redirects

When a host loads a cross-origin iframe, the iframe host is recorded as a subframe under that host. When a host redirects to another host, this redirect relationship is also recorded by the [WebResourceLoadStatisticsStore](https://github.com/WebKit/webkit/blob/master/Source/WebKit/UIProcess/WebResourceLoadStatisticsStore.h).

This statistics collection is not affected by any combination of:
- **Original subframe location** - a subframe can start on the same origin and redirect to a "tracker", or it can start on the "tracker" and redirect back to the same origin.
- **Nested iframes** - a subframe on the same origin can load a nested iframe that redirects between hosts.
- **Iframe sandboxing** - a subframe can have the `sandbox="allow-scripts"` attribute, but it will have no effect on stats collection.

**Test References**:
- [non-sandboxed-iframe-redirect-ip-to-localhost-to-ip](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/non-sandboxed-iframe-redirect-ip-to-localhost-to-ip.html)
- [non-sandboxed-iframe-redirect-localhost-to-ip-to-localhost](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/non-sandboxed-iframe-redirect-localhost-to-ip-to-localhost.html)
- [non-sandboxed-nesting-iframe-with-non-sandboxed-iframe-redirect-ip-to-localhost-to-ip](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/non-sandboxed-nesting-iframe-with-non-sandboxed-iframe-redirect-ip-to-localhost-to-ip.html)
- [non-sandboxed-nesting-iframe-with-non-sandboxed-iframe-redirect-localhost-to-ip-to-localhost](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/non-sandboxed-nesting-iframe-with-non-sandboxed-iframe-redirect-localhost-to-ip-to-localhost.html)
- [non-sandboxed-nesting-iframe-with-sandboxed-iframe-redirect-ip-to-localhost-to-ip](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/non-sandboxed-nesting-iframe-with-sandboxed-iframe-redirect-ip-to-localhost-to-ip.html)
- [non-sandboxed-nesting-iframe-with-sandboxed-iframe-redirect-localhost-to-ip-to-localhost](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/non-sandboxed-nesting-iframe-with-sandboxed-iframe-redirect-localhost-to-ip-to-localhost.html)
- [sandboxed-iframe-redirect-ip-to-localhost-to-ip](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/sandboxed-iframe-redirect-ip-to-localhost-to-ip.html)
- [sandboxed-iframe-redirect-localhost-to-ip-to-localhost](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/sandboxed-iframe-redirect-localhost-to-ip-to-localhost.html)
- [sandboxed-nesting-iframe-with-non-sandboxed-iframe-redirect-ip-to-localhost-to-ip](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/sandboxed-nesting-iframe-with-non-sandboxed-iframe-redirect-ip-to-localhost-to-ip.html)
- [sandboxed-nesting-iframe-with-non-sandboxed-iframe-redirect-localhost-to-ip-to-localhost](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/sandboxed-nesting-iframe-with-non-sandboxed-iframe-redirect-localhost-to-ip-to-localhost.html)
- [sandboxed-nesting-iframe-with-sandboxed-iframe-redirect-ip-to-localhost-to-ip](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/sandboxed-nesting-iframe-with-sandboxed-iframe-redirect-ip-to-localhost-to-ip.html)
- [sandboxed-nesting-iframe-with-sandboxed-iframe-redirect-localhost-to-ip-to-localhost](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/sandboxed-nesting-iframe-with-sandboxed-iframe-redirect-localhost-to-ip-to-localhost.html)

### Hardcoded SSO exceptions

#### Dow Jones

> :warning: Removed in [#188756](https://bugs.webkit.org/show_bug.cgi?id=188756) - [Remove experimental affiliated domain code now that StorageAccess API is available](https://github.com/WebKit/webkit/commit/d91eb617805ac70fc01f5aa3201086c3b2c15d56).

Statistics are not logged when navigating between pages on the same domain. There is also some hardcoded logic for ignoring navigations between *associated domains* - this stemmed from [#174661 May get frequently logged out of wsj.com](https://bugs.webkit.org/show_bug.cgi?id=174661) and is a hardcoded list of domain associations that are exempt from statistics collection.

[The original commit](https://github.com/WebKit/webkit/commit/0a88fc73392d26beeaa996b9364c359984fba887) associates `dowjones.com`, `wsj.com`, `barrons.com`, `marketwatch.com`, and `wsjplus.com`, which form a classic SSO scenario - WJS, Barrons, and the others have a sign-in link that redirects to a shared *sso.accounts.dowjones.com* account chooser.

The code has since moved to [ResourceLoadStatistics.cpp](https://github.com/WebKit/webkit/blob/e460e444bb89d90a2ba96087f1c94ae0360f778a/Source/WebCore/loader/ResourceLoadStatistics.cpp#L361-L373), and Dow Jones is still the only exception in the list.

#### Adobe

> :warning: Removed in [#188710](https://bugs.webkit.org/show_bug.cgi?id=188710) - [Remove Adobe SSO exception now that StorageAccess API is available](https://github.com/WebKit/webkit/commit/a0a4879d0006cf97b5af06bcf72a19f874dda252).

Another more general approach to whitelisting SSO scenarios is from [#174533 Can no longer log in on abc.go.com](https://bugs.webkit.org/show_bug.cgi?id=174533), where *abc.go.com* and others used *sp.auth.adobe.com* as their IdP and it became marked as prevalent. In this approach, there are no associated domains - a host is whitelisted from any statistics collection and cannot be marked as prevalent.

More detail [can be found in the commit](https://github.com/WebKit/webkit/commit/6826121dd1515bd116f2f318aa3ddaec6773fedf). Adobe is [still the only host in the whitelist](https://github.com/WebKit/webkit/blob/e99033fdebb296529f1c786bb50cb9494176a841/Source/WebCore/loader/ResourceLoadObserver.cpp#L108-L114). Note - this whitelist is [tied to the needsSiteSpecificQuirks setting](https://github.com/WebKit/webkit/commit/e272023a96c1cc9e67ecdc99ed72d7a33f05f1d5), which means that it can be disabled.

## Classification

### Prevalence

A prevalent resource is a domain that has demonstrated the ability to track users cross-site. It's another name for *tracker*, with the extra nuance that it's unknown what the domain does with its ability to track ([#182664](https://bugs.webkit.org/show_bug.cgi?id=182664)).

When statistics for a resource have changed, they're run through a classifier that categorizes the resource as *prevalent* or *not-prevalent*. The prevalence level, combined with user interaction signals, determine what actions to take against the site.

There are two classifiers in the WebKit codebase:
- *Vector Threshold* - The older, simpler model that is used when Core Prediction is not enabled.
- *Core Prediction* - The newer, machine learning based model.

#### Vector Threshold

Vector Threshold is the classifier that's used when Core Prediction is not enabled - notably, in all [ResourceLoadStatistics tests](https://github.com/WebKit/webkit/tree/master/LayoutTests/http/tests/resourceLoadStatistics). It uses raw counts and a simple vector length algorithm to classify prevalence into three buckets - *Low* (not-prevalent), *High* (prevalent), and *Very High* (very prevalent).

```bash
a = subresourceUnderTopFrameOriginsCount
b = subresourceUniqueRedirectsToCount
c = subframeUnderTopFrameOriginsCount
d = topFrameUniqueRedirectsToCount

vectorLength(x, y, z) = sqrt(x^2 + y^2 + z^2)

if vectorLength(a, b, c) > 30
  # Prevalence is Very High
else if currentPrevalence == High
  || a > 3
  || b > 3
  || c > 3
  || d > 3
  || vectorLength(a, b, c) > 3
  # Prevalence is High
else
  # Prevalence is Low
```

The important levels are *Low* (not-prevalent) and *High/Very High* (prevalent). The distinction for *Very High* is only used to report back to Apple via telemetry ([#183218](https://bugs.webkit.org/show_bug.cgi?id=183218)). Some examples of statistics that would classify a resource as prevalent:
- A domain is redirected to by more than 3 other domains
- A domain is loaded in a subframe by more than 3 other domains

**Source References**

- [ResourceLoadStatisticsClassifer.cpp](https://github.com/WebKit/webkit/blob/master/Source/WebKit/Platform/classifier/ResourceLoadStatisticsClassifier.cpp)
- [#183218](https://bugs.webkit.org/show_bug.cgi?id=183218) - [Add a second tier of prevalence to facilitate telemetry on very prevalent domains](https://github.com/WebKit/webkit/commit/5f5ea95f44fdde40b7f2b9ef978637c60321417e)

**Test References**

- [classify-as-non-prevalent-based-on-mixed-statistics](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/classify-as-non-prevalent-based-on-mixed-statistics.html)
- [classify-as-non-prevalent-based-on-sub-frame-under-top-frame-origins](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/classify-as-non-prevalent-based-on-sub-frame-under-top-frame-origins.html)
- [classify-as-non-prevalent-based-on-subresource-under-top-frame-origins](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/classify-as-non-prevalent-based-on-subresource-under-top-frame-origins.html)
- [classify-as-non-prevalent-based-on-subresource-unique-redirects-to](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/classify-as-non-prevalent-based-on-subresource-unique-redirects-to.html)
- [classify-as-prevalent-based-on-mixed-statistics](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/classify-as-prevalent-based-on-mixed-statistics.html)
- [classify-as-prevalent-based-on-sub-frame-under-top-frame-origins](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/classify-as-prevalent-based-on-sub-frame-under-top-frame-origins.html)
- [classify-as-prevalent-based-on-subresource-under-top-frame-origins](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/classify-as-prevalent-based-on-subresource-under-top-frame-origins.html)
- [classify-as-prevalent-based-on-subresource-unique-redirects-to](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/classify-as-prevalent-based-on-subresource-unique-redirects-to.html)
- [classify-as-prevalent-based-on-top-frame-unique-redirects-to](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/classify-as-prevalent-based-on-top-frame-unique-redirects-to.html)
- [classify-as-very-prevalent-based-on-mixed-statistics](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/classify-as-very-prevalent-based-on-mixed-statistics.html)

#### Core Prediction

Core Prediction is an [SVM Classifier](https://en.wikipedia.org/wiki/Support_vector_machine) that calculates prevalence using a machine learning model trained on three features:
- *subresourceUnderTopFrameOriginCounts*
- *subresourceUniqueRedirectsToCount*
- *subframeUnderTopFrameOriginsCount*

Aside from the underlying algorithm, there are some other key differences from Vector Threshold:
- It does not have the granularity of Vector Threshold's *Low*, *High*, and *Very High*. Core Prediction only returns true if prevalent, false if not.
- It does not make use of top frame redirects. This is either a missing feature, or this means that Vector Threshold will always be used in some code paths to be able to catch first-party bounce trackers.

**Source References**
- [ResourceLoadStatisticsClassifierCocoa.cpp](https://github.com/WebKit/webkit/blob/master/Source/WebKit/Platform/classifier/cocoa/ResourceLoadStatisticsClassifierCocoa.cpp)
- [#168347](https://bugs.webkit.org/show_bug.cgi?id=168347) - [Add alternate classification method](https://github.com/WebKit/webkit/commit/8842d53a15a751947ddc3f578a1b75b4163e8335)

### User interaction

User interaction on a resource is used to determine how to handle sites that are marked as prevalent. It is categorized into three buckets: *no user interaction*, *recent user interaction*, and *non-recent user interaction*.

Even if a user interaction event is triggered in a subframe, it is always recorded as an interaction on the top domain. From [#174120](https://bugs.webkit.org/show_bug.cgi?id=174120):

> No matter where on a webpage a user interacts, the interaction should always be recorded for the top document. Users have no way of understanding what a cross-origin iframe is and they have no visual way to tell content from different origins apart.

To allow users to maintain sessions in embedded third-party services they rarely use as first party, Apple introduced the [Storage Access API](https://webkit.org/blog/8124/introducing-storage-access-api/). From the **User Prompt for the Storage Access API** section in [ITP 2.0](https://webkit.org/blog/8311/intelligent-tracking-prevention-2-0/):

> Successful use of the Storage Access API now counts as user interaction with the *third-party* and refreshes the 30 days of use before ITP purges the third-party’s website data. By successful use we mean the user was either prompted right now and chose “Allow,” or had previously chosen “Allow.” The fact that successful Storage Access API calls now count as user interaction allows users to stay logged into services they rarely use as first party but keep using as embedded third parties.

User interaction is reported immediately to the UI process, which means that websites can immediately get out of a blocked state instead of waiting for the prevalence digest cycle to run.

Any *handled* event is a candidate for user interaction, where a *handled* event is an event that Safari recognizes and internally delegates. An example of an *unhandled* event that will not be classified as user interaction is an unfinished key chord sequence, i.e. pressing `meta` with no following keypress.

**Source References**
- [#174120](https://bugs.webkit.org/show_bug.cgi?id=174120) - [User interaction should always go to the top document](https://github.com/WebKit/webkit/commit/9795914c210fff5c094844231c225eb90d3968cc)
- [#178183](https://bugs.webkit.org/show_bug.cgi?id=178183) - [CMD+Q keyboard shortcuts are treated as user interaction with page](https://github.com/WebKit/webkit/commit/56f28105ab9fd8596273b3edc1031d3924d7c977)
- [#175090](https://bugs.webkit.org/show_bug.cgi?id=175090) - [Report user interaction immediately, but only when needed](https://github.com/WebKit/webkit/commit/75153582978a98ec82a2741f46f79401415b9526)

**Test References**
- [prevalent-resource-handled-keydown](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/prevalent-resource-handled-keydown.html)
- [prevalent-resource-unhandled-keydown](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/prevalent-resource-unhandled-keydown.html)
- [user-interaction-in-cross-origin-sub-frame](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/user-interaction-in-cross-origin-sub-frame.html)
- [user-interaction-only-reported-once-within-short-period-of-time](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/user-interaction-only-reported-once-within-short-period-of-time.html)
- [user-interaction-reported-after-website-data-removal](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/user-interaction-reported-after-website-data-removal.html)

### Bounce trackers and redirect collusion

ITP provides protection against bounce trackers, and can detect tracker collusion.

From [Intelligent Tracking Prevention 2.0](https://webkit.org/blog/8311/intelligent-tracking-prevention-2-0/):

> ITP 2.0 has the ability to detect when a domain is solely used as a “first party bounce tracker,” meaning that it is never used as a third party content provider but tracks the user purely through navigational redirects.

This translates to marking prevalent any resource that directly or indirectly redirects to a prevalent resource, where redirects are defined as [having a 3xx status code](https://github.com/WebKit/webkit/blob/42144f7f8aaf261d1ed745b0e36f8f1df9539a0d/Source/WebCore/loader/ResourceLoadObserver.cpp#L124). This applies to both top frame and subframe redirects, and is retroactive - not only are future redirects to this resource classified as prevalent, but also all resources that *have redirected to this resource in the past*.

**Direct redirects**

In the following example, **A** is prevalent and **B** is not prevalent. After **B** redirects to **A**, it's marked as prevalent.

```bash
# A is prevalent. B is not prevalent.
B -> A
# A,B are prevalent
```

**Tracker collusion**

Redirects to multiple resources that end in a prevalent resource is described as *tracker collusion*, with the graph of redirect nodes described as a *collusion graph*. All resources in the collusion graph will be marked as prevalent through recursion - the closest resource will be marked as prevalent, the resource that redirected to it will be marked as prevalent, and so on up to a max limit of [50 cycles](https://github.com/WebKit/webkit/commit/27ad4162c4b30e763c24d60326df153675d5ef33#diff-9d3147a36bc4a82e4bb4877a02e9f85bR51).

In the following example, **A** is prevalent, and **B**, **C**, **D** are not prevalent. In the first pass, **B** is marked as prevalent. In the next pass, **C** is marked as prevalent. And in the final pass, **D** is marked as prevalent, which completes the collusion graph.

```bash
# A is prevalent. B,C,D are not prevalent.
D -> C -> B -> A
# A,B,C,D are prevalent
```

**Source References**

- [#186627](https://bugs.webkit.org/show_bug.cgi?id=186627) - [Shortcut classification for redirect to prevalent resource](https://github.com/WebKit/webkit/commit/e28db7d0cc2fc612fdf642f3d00be47e93b3a0cf)
- [#182664](https://bugs.webkit.org/show_bug.cgi?id=182664) - [Classify resources as prevalent based on redirects to other prevalent resources](https://github.com/WebKit/webkit/commit/27ad4162c4b30e763c24d60326df153675d5ef33#diff-9d3147a36bc4a82e4bb4877a02e9f85bR51)

**Test References**

- [classify-as-prevalent-based-on-subresource-redirect-collusion](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/classify-as-prevalent-based-on-subresource-redirect-collusion.html)
- [classify-as-prevalent-based-on-subresource-redirect-to-prevalent](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/classify-as-prevalent-based-on-subresource-redirect-to-prevalent.html)
- [classify-as-prevalent-based-on-top-frame-redirect-collusion](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/classify-as-prevalent-based-on-top-frame-redirect-collusion.html)
- [classify-as-prevalent-based-on-top-frame-redirect-to-prevalent](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/classify-as-prevalent-based-on-top-frame-redirect-to-prevalent.html)

## Effects of classification

### Access to cookies in a third-party context

**Definitions**

- **First-party cookie** - A cookie that is set and read on the primary domain
- **Third-party context** - Cross-origin subframe
- **Third-party cookie** - First-party cookies that are accessed in a third-party context

When a cross-origin iframe is loaded for a resource that's classified as prevalent, access to third-party cookies is blocked.

This had been the intention before [ITP 2.0](https://webkit.org/blog/8311/intelligent-tracking-prevention-2-0/), but previous versions had more leeway around enforcement:
- There was a 1-day window after user interaction where cookies could be accessed in a third-party context for a prevalent resource (see [ITP 1.1](https://webkit.org/blog/8142/intelligent-tracking-prevention-1-1/) and [ITP 1.0](https://webkit.org/blog/7675/intelligent-tracking-prevention/)).
- There was a separate class of *partitioned cookies* that gave cross-origin iframes a way to read and write cookies keyed to the subframe and top frame, which worked as long as the user had interacted with the third-party resource in the last 30 days. Partitioned cookies were recently removed [in this commit](https://github.com/WebKit/webkit/commit/e460e444bb89d90a2ba96087f1c94ae0360f778a).

With ITP 2.0, non-prevalent resources can read and write third-party cookies. Prevalent resources cannot, unless they request storage access via the [Storage Access API](https://webkit.org/blog/8124/introducing-storage-access-api/).

**Source References**
- [#188109](https://bugs.webkit.org/show_bug.cgi?id=188109) - [Remove partitioned cookies for reduced complexity, lower memory footprint, and ability to support more platforms](https://github.com/WebKit/webkit/commit/e460e444bb89d90a2ba96087f1c94ae0360f778a)

**Test References**
- [cookies-with-and-without-user-interaction](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/cookies-with-and-without-user-interaction.html)
- [non-prevalent-resources-can-access-cookies-in-a-third-party-context](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/non-prevalent-resources-can-access-cookies-in-a-third-party-context.html)

### Automatic cookie access for popups

There is a temporary compatibility fix to access first-party cookies from a third-party subframe when a popup window to the third-party is opened on the same page.

An example scenario:
1. User clicks a social login button on rp.com
2. Popup window is opened to idp.com, which has previously been marked prevalent
3. User interacts with the popup to login to idp.com, which sets a first-party session cookie
4. The popup closes and sends a postMessage to window.opener, rp.com
5. The main page opens a cross-origin iframe to idp.com. Because of the compatibility fix, this subframe has access to the first-party session cookie that was previously set.

The popup origin must have had recent user interaction to grant access in the third-party subframe. If there is no recent user interaction, the first-party cookie is not accessible.

From the **Automatic storage access for popups** section in [ITP 2.0](https://webkit.org/blog/8311/intelligent-tracking-prevention-2-0/):
> Many federated logins send authentication tokens in URLs or through the postMessage API, both of which work fine under ITP 2.0. However, some federated logins use popups and then rely on third-party cookie access once the user is back on the opener page. Some instances of this latter category stopped working under ITP 2.0 since domains with tracking abilities are permanently partitioned.
>
> Our temporary compatibility fix is to detect such popup scenarios and **automatically forward storage access for the third party under the opener page**. Since popups require user interaction, the third party could just as well had called the Storage Access API instead.
>
> **Developer Advice:** If you provide federated login services, we encourage you to first call the Storage Access API to get cookie access and only do a popup to log the user in or acquire specific consent. The Storage Access API provides a better user experience without new windows and navigations. We’d also like to stress that **the compatibility fix for popups is a temporary one**. Longterm, your only option will be to call the Storage Access API.

**Source Resources**

- [#183620](https://bugs.webkit.org/show_bug.cgi?id=183620) - [Immediately forward cookie access for domains with previous user interaction when there's an opener document](https://github.com/WebKit/webkit/commit/bbc57631e902c47b884d0fbf51935f5bdb4ac528)

**Test Resources**
- [deny-storage-access-under-opener](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/deny-storage-access-under-opener.html)
- [grant-storage-access-under-opener](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/grant-storage-access-under-opener.html)

### Purging cookies

If a resource is prevalent, it is subject to having its first-party cookies purged depending on when the user has last interacted with the domain. If the user has never interacted with the prevalent domain, or has not interacted within the *TimeToLiveUserInteraction* ([last 30 days](https://github.com/WebKit/webkit/blob/e460e444bb89d90a2ba96087f1c94ae0360f778a/Source/WebKit/UIProcess/Cocoa/ResourceLoadStatisticsMemoryStoreCocoa.mm#L38-L39)), first-party cookies will be deleted.

```bash
# Setup: idp.com is prevalent

1. User visits idp.com, interacts with login form, new SID cookie.
2. After 15 days, user redirects to idp.com, no user interaction. SID still exists.

# -- after 30 days with no user interaction, SID cookie is purged --

3. User redirects to idp.com after 30 days, SID is not available.
```

When cookies are purged, they're deleted and can no longer be accessed even if the resource later becomes non-prevalent. Non-prevalent resources never get their first-party cookies purged, regardless of user interaction.

**Source References**
- [#188109](https://bugs.webkit.org/show_bug.cgi?id=188109) - [Remove partitioned cookies](https://github.com/WebKit/webkit/commit/e460e444bb89d90a2ba96087f1c94ae0360f778a)
- [#167474](https://bugs.webkit.org/show_bug.cgi?id=167474) - [Get the right website data store and introduce timeout for user interaction](https://github.com/WebKit/webkit/commit/a1514afcbac711363e142386dd67c89b87deee77)
- [#172155](https://bugs.webkit.org/show_bug.cgi?id=172155) - [Grandfather domains for existing records](https://github.com/WebKit/webkit/commit/2a5e5a0dd7fde384f426469e5747dbd358697b5c)

**Test References**
- [cookie-deletion](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/cookie-deletion.html)
- [grandfathering](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/grandfathering.html)
- [non-prevalent-resource-with-user-interaction](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/non-prevalent-resource-with-user-interaction.html)
- [non-prevalent-resource-without-user-interaction](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/non-prevalent-resource-without-user-interaction.html)
- [prevalent-resource-with-user-interaction-timeout](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/prevalent-resource-with-user-interaction-timeout.html)
- [prevalent-resource-with-user-interaction](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/prevalent-resource-with-user-interaction.html)
- [prevalent-resource-without-user-interaction](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/prevalent-resource-without-user-interaction.html)

### First-party cookies and redirects

Access to first-party cookies in top level navigation redirects is never blocked. This is good for SSO redirects which rely on an existing session in the IdP - even if the domain is prevalent, the existing first-party session cookie will always be available.

However, it is counterintuitive - once [the commit to not block first party cookies on redirects](https://github.com/WebKit/webkit/commit/e89a3208aa0dfad05c8a609b3fb0fe6242d35cac) was merged, protection against bounce trackers became much more limited. Bounce trackers will be able to set and read cookies on redirects (which is what they want to do), and are only blocked when reading cookies from a third-party context.

**Source References**
- [#177394](https://bugs.webkit.org/show_bug.cgi?id=177394) - [Block cookies for prevalent resources without user interaction](https://github.com/WebKit/webkit/commit/56cbaf543289f0919548e82f69eed0bd0609ebf5)
- [#184948](https://bugs.webkit.org/show_bug.cgi?id=184948) - [Don't Block First Party Cookies on Redirects](https://github.com/WebKit/webkit/commit/e89a3208aa0dfad05c8a609b3fb0fe6242d35cac)

**Test References**
- [add-blocking-to-redirect](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/add-blocking-to-redirect.html)
- [do-not-block-top-level-navigation-redirect](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/do-not-block-top-level-navigation-redirect.html)
- [remove-blocking-in-redirect](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/remove-blocking-in-redirect.html)

### Stripping Referer Header

When making CORS requests or redirecting to a prevalent resource, the *Referer* header is restricted - only the origin is sent. This limits the user data that can be collected by a domain that's known to be able to track users.

Example:
```bash
# Referer header that's sent to a non-prevalent resource
http://some-store.com/some-category/item123

# Referer header that's sent to a prevalent resource
http://some-store.com/
```

**Source References**

- [#182559](https://bugs.webkit.org/show_bug.cgi?id=182559) - [Restrict Referer to just the origin for third parties in private mode and third parties ITP blocks cookies for in regular mode](https://github.com/WebKit/webkit/commit/bddbe342e879ee67a6419fba601a8b863bf3c89a?diff=unified)

**Test References**

- [strip-referrer-to-origin-for-prevalent-subresource-redirects](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/strip-referrer-to-origin-for-prevalent-subresource-redirects.html)
- [strip-referrer-to-origin-for-prevalent-subresource-requests](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/strip-referrer-to-origin-for-prevalent-subresource-requests.html)

## Storage access

When a cross-origin iframe is loaded from a prevalent domain, access to first-party cookies is blocked by default. To support embedded third-party content that use first-party cookies for authentication, Apple [introduced the Storage Access API](https://webkit.org/blog/8124/introducing-storage-access-api/). This API provides a way for cross-origin iframes to request access to their first-party cookies *when processing a user gesture such as a tap or a click*.

The problem statement, Apple's proposed solution, and community feedback can be found in their [WHATWG Proposal](https://github.com/whatwg/html/issues/3338).

### hasStorageAccess

The `document.hasStorageAccess()` method can be called at any time to check whether access is already granted, and does not require a user gesture. If it is `true`, the caller can read and write first-party cookies in a third-party context.

**Example**
```javascript
const hasAccess = await document.hasStorageAccess();
if (hasAccess) {
  // Can access first-party cookies
} else {
  // Cannot access first-party cookies
}
```

Before a call to `requestStorageAccess()`, the response from `hasStorageAccess()` depends only on the prevalence of the domain, and not whether there has been previous user interaction:

| Prevalence | Has had user interaction? | hasStorageAccess default response |
| -- | -- | -- |
| prevalent | yes | `false` |
| prevalent | no | `false` |
| not-prevalent | yes | `true` |
| not-prevalent | no | `true` |

**Test Resources**
- [has-storage-access-from-prevalent-domain-with-user-interaction](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/has-storage-access-from-prevalent-domain-with-user-interaction.html)

### requestStorageAccess

The `document.requestStorageAccess()` method returns a promise that is resolved if storage access was granted and is rejected if access was denied. It *must* be called within the context of a user gesture, like a tap or a click - if it is not, it will immediately reject before prompting the user for consent.

**Example**
```html
<script>
function makeRequestWithUserGesture() {
  document.requestStorageAccess().then(
    () => {
      // Storage access is granted:
      // - Access to first-party cookies is available
      // - Subsequent calls to hasStorageAccess() will return true
    },
    () => {
      // Storage access is denied
    }
  );
}
</script>
<button onclick="makeRequestWithUserGesture()"></button>
```

The algorithm for `requestStorageAccess` is laid out in full in the [Storage Access API WHATWG Proposal](https://github.com/whatwg/html/issues/3338):

**#1: If the document already has been granted access, resolve.**

If requesting access for a prevalent domain where the user has already given consent for this subframe, this will resolve.

Access must be requested for each third-party resource on the page - consent given to one subframe will not implicitly transfer to another subframe, even if that subframe is on the same domain. However, if the user allows access, their choice is persisted for that domain and the next `requestStorageAccess` call will not prompt the user.

From the **User Prompt for the Storage Access API** section in [ITP 2.0](https://webkit.org/blog/8311/intelligent-tracking-prevention-2-0/):
> If the user allows access, their choice is persisted. If the user declines, their choice is not persisted which allows them to change their mind if they at a later point tap a similar embedded widget that calls the Storage Access API.

**#2: If the document has a null origin, reject.**

By default, a [sandboxed iframe](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe#attr-sandbox) will have a unique, or null, origin. If this is the case, `requestStorageAccess` immediately rejects because there are no first-party cookies to return. To get around this, a sandboxed iframe *must* have the `allow-same-origin` token, which allows the content to be treated from its original, third-party origin.

**#3: If the document's frame is the main frame, resolve.**

Resolve immediately if `requestStorageAccess` is called in the top frame, which will always have access to its own first-party cookies.

**#4: If the sub frame's origin is equal to the main frame's, resolve.**

Resolve immediately if `requestStorageAccess` is called in a subframe that is on the same origin as the main frame - ITP only affects access to cookies in a cross-origin context.

For ITP, Safari has a looser definition of "origin" than described in the [same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy). It is defined as *eTLD+1*, where the effective top level domain "eTLD" is .com, .co.uk, etc. This means that resources that share the same *eTLD+1* are not affected by ITP, and will be able to access cookies even if the domain is marked as prevalent.

```javascript
// Subdomains are not affected by ITP 2.0
etld_1('login.social.com')   == 'social.com'; // true
etld_1('email.social.com')   == 'social.com'; // true
etld_1('foo.bar.social.com') == 'social.com'; // true
```

**#5: If the sub frame is not sandboxed, skip to step 7.**

**#6: If the sub frame doesn't have the token "allow-storage-access-by-user-activation", reject.**

The default for [sandboxed iframes](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe#attr-sandbox) is to not have access to the new storage functions. Apple introduced a new token `allow-storage-access-by-user-activation` to opt-in to these storage functions, which means that a sandboxed iframe must have at least these three tokens:
- `allow-scripts` - allow scripts to run, which is a prerequisite to calling the storage methods
- `allow-storage-access-by-user-activation` - access to the storage methods
- `allow-same-origin` - use the original origin for the frame, covered in #2

```html
<iframe
  src="https://thirdpartyorigin.com/example"
  sandbox="allow-storage-access-by-user-activation allow-same-origin allow-scripts" />
```

**#7: If the sub frame's parent frame is not the top frame, reject.**

Nested iframes that have passed the previous same-origin and main document checks are never allowed storage access, even if they have the right sandbox tokens, etc. A nested iframe is a frame whose parent frame is not the top frame - i.e. a page embeds an iframe, which embeds another *nested* iframe.

**#8: If the browser is not processing a user gesture, reject.**

A call to `document.requestStorageAccess()` must be made within the context of a user gesture, like a tap or a click.

```html
<!-- This is valid -->
<script>
function makeRequestWithUserGesture() {
  document.requestStorageAccess().then(/* this is valid */);
}
</script>
<button onclick="makeRequestWithUserGesture()"></button>

<!-- This will immediately reject -->
<script>
function makeRequestWithoutUserGesture() {
  document.requestStorageAccess().then(/* this will always reject */);
}
</script>
<body onload="makeRequestWithoutUserGesture()"></body>
```

**#9: Check any additional rules that the browser has. Examples: Whitelists, blacklists, on-device classification, user settings, anti-clickjacking heuristics, or prompting the user for explicit permission. Reject if some rule is not fulfilled.**

ITP classification is used in this step.

If requesting access for a non-prevalent domain, this will resolve. This will resolve positively regardless of whether or not there has been previous user interaction on the domain.

If the domain is prevalent and *there has been no previous user interaction on the domain*, this [will immediately reject](https://github.com/WebKit/webkit/blob/e460e444bb89d90a2ba96087f1c94ae0360f778a/LayoutTests/http/tests/storageAccess/request-and-grant-access-cross-origin-sandboxed-iframe-from-prevalent-domain-without-user-interaction.html#L7). There's no chance for prompting the user if the user has not previously interacted with the prevalent domain in a first-party context.

If the domain is prevalent and there has been previous user interaction, Safari will show the ITP 2.0 storage consent prompt. If the user chooses to not allow the domain to track their activity, this will reject.

**#10: Grant the document access to cookies and store that fact for the purposes of future calls to hasStorageAccess() and requestStorageAccess().**

**Source References**
- [Document::hasStorageAccess](https://github.com/WebKit/webkit/blob/c187113ceea90299d18579d2b5d16203800b463e/Source/WebCore/dom/Document.cpp#L7400)
- [Document::requestStorageAccess](https://github.com/WebKit/webkit/blob/c187113ceea90299d18579d2b5d16203800b463e/Source/WebCore/dom/Document.cpp#L7439)
- [#176939](https://bugs.webkit.org/show_bug.cgi?id=176939) - [Deny access to nested iframes](https://github.com/WebKit/webkit/commit/6f74c8f20226692dcc80f18f9590b0089dfa13a3)
- [#176944](https://bugs.webkit.org/show_bug.cgi?id=176944) - [https://bugs.webkit.org/show_bug.cgi?id=176944](https://github.com/WebKit/webkit/commit/c187113ceea90299d18579d2b5d16203800b463e)

**Test References**
- [request-and-grant-access-cross-origin-non-sandboxed-iframe](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/request-and-grant-access-cross-origin-non-sandboxed-iframe.html)
- [request-and-grant-access-cross-origin-sandboxed-iframe-from-prevalent-domain-with-user-interaction](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/request-and-grant-access-cross-origin-sandboxed-iframe-from-prevalent-domain-with-user-interaction.html)
- [request-and-grant-access-cross-origin-sandboxed-iframe-from-prevalent-domain-without-user-interaction](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/request-and-grant-access-cross-origin-sandboxed-iframe-from-prevalent-domain-without-user-interaction.html)
- [request-and-grant-access-cross-origin-sandboxed-iframe](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/request-and-grant-access-cross-origin-sandboxed-iframe.html)
- [request-and-grant-access-cross-origin-sandboxed-nested-iframe](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/request-and-grant-access-cross-origin-sandboxed-nested-iframe.html)
- [request-storage-access-cross-origin-sandboxed-iframe-without-allow-token](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/request-storage-access-cross-origin-sandboxed-iframe-without-allow-token.html)
- [request-storage-access-cross-origin-sandboxed-iframe-without-user-gesture](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/request-storage-access-cross-origin-sandboxed-iframe-without-user-gesture.html)
- [request-storage-access-same-origin-sandboxed-iframe-without-allow-token](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/request-storage-access-same-origin-sandboxed-iframe-without-allow-token.html)
- [request-storage-access-same-origin-iframe](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/request-storage-access-same-origin-iframe.html)
- [request-storage-access-same-origin-sandboxed-iframe](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/request-storage-access-same-origin-sandboxed-iframe.html)
- [request-storage-access-top-frame](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/request-storage-access-top-frame.html)

### Lifetime of storage access

From the **Access Removal** section of the [Storage Access API WHATWG proposal](https://github.com/whatwg/html/issues/3338#issue-287518824):

> Storage access is granted for the life of the document and as long as the document's frame is attached to the DOM. This means:
> - Access is removed when the sub frame navigates.
> - Access is removed when the sub frame is detached from the DOM.
> - Access is removed when the top frame navigates.
> - Access is removed when the webpage goes away, such as a tab close.

Some concrete examples:
- If a subframe has been granted storage access and the top frame changes the `src` of that subframe, the subframe loses storage access.
- If there are two subframes for the same cross-origin domain on the same page and one has been granted storage access, storage access is not automatically transferred to the other frame. The other frame must also request storage access.
- If a subframe has been granted storage access and it's moved in the DOM (i.e. by appending it to another element), it loses storage access. This is an example of detaching it from the DOM.
- If a subframe has been granted storage access and it internally navigates to a different page (i.e. in a redirect scenario), it loses storage access.

**Source Resources**
- [#180728](https://bugs.webkit.org/show_bug.cgi?id=180728) - [Make DocumentLoader::willSendRequest() and WebFrameLoaderClient::detachedFromParent2() tell the network process to get rid of any sub frame access entries](https://github.com/WebKit/webkit/commit/7683422de0b8f92828db7cb18dadce6b8656450b)
- [#180679](https://bugs.webkit.org/show_bug.cgi?id=180679) - [Implement frame-specific access in the network storage session layer](https://github.com/WebKit/webkit/commit/6ed20e67087368b5edd723fd177c51b68e98a8cc)

**Test Resources**
- [request-and-grant-access-cross-origin-sandboxed-iframe-from-prevalent-domain-with-user-interaction-and-access-from-right-frame](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/request-and-grant-access-cross-origin-sandboxed-iframe-from-prevalent-domain-with-user-interaction-and-access-from-right-frame.html)
- [request-and-grant-access-cross-origin-sandboxed-iframe-from-prevalent-domain-with-user-interaction-but-access-from-wrong-frame](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/request-and-grant-access-cross-origin-sandboxed-iframe-from-prevalent-domain-with-user-interaction-but-access-from-wrong-frame.html)
- [request-and-grant-access-then-detach-should-not-have-access](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/request-and-grant-access-then-detach-should-not-have-access.html)
- [request-and-grant-access-then-navigate-should-not-have-access](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/request-and-grant-access-then-navigate-should-not-have-access.html)

### Exceptions

#### Popups after requestStorageAccess

Both `document.requestStorageAccess()` and `window.open` depend on being called in the context of a user gesture. For cross-origin iframes that open a popup window, this presents a problem - `requestStorageAccess` is async, so the user gesture context wouldn't normally be set for its callback handlers. [This commit](https://github.com/WebKit/webkit/commit/2cf19661a4ce945c2ff7dbc8560654613ffdd8b2) provides an override to allow popups to be opened after the resolution of `requestStorageAccess`.

```html
<script>
function makeRequestWithUserGesture() {
  document.requestStorageAccess().then(
    () => {
      // User gesture still intact, even after the async callback
      window.open('test-window.html', 'test window');
    }
  );
}
</script>
<button onclick="makeRequestWithUserGesture()"></button>
```

**Source References**
- [#185615](https://bugs.webkit.org/show_bug.cgi?id=185615) - [Allow documents that have been granted storage access to also do a popup](https://github.com/WebKit/webkit/commit/2cf19661a4ce945c2ff7dbc8560654613ffdd8b2)

**Test References**
- [request-and-grant-access-cross-origin-non-sandboxed-iframe-pop-window](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/storageAccess/request-and-grant-access-cross-origin-non-sandboxed-iframe-pop-window.html)

## Miscellaneous

### Resetting ITP statistics and classification

All ITP statistics and classification is stored locally on device, and can be cleared by using the *remove all website data* feature.

**Source References**
- [#169448](https://bugs.webkit.org/show_bug.cgi?id=169448) - [Remove statistics data as part of full website data removal](https://github.com/WebKit/webkit/commit/22b264e1ebfba0aa035d0a3bb5ae25491b1c6d60)
- [#171584](https://bugs.webkit.org/show_bug.cgi?id=171584) - [Remove all statistics for modifiedSince website data removals](https://github.com/WebKit/webkit/commit/ab4b1afba9672e340770078bdd287980fcb0913f)

**Test References**
- [clear-in-memory-and-persistent-store-one-hour](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/clear-in-memory-and-persistent-store-one-hour.html)
- [clear-in-memory-and-persistent-store](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/clear-in-memory-and-persistent-store.html)

### Pruning Statistics

There is a cap on the number of statistics records that are saved in the statistics store. When this limit is exceeded, the store is pruned in order of least importance:
- Non-prevalent resources without user interaction
- Prevalent resources without user interaction
- Non-prevalent resources with user interaction
- Prevalent resources with user interaction

**Source References**
- [#174215](https://bugs.webkit.org/show_bug.cgi?id=174215) - [Prune statistics in orders of importance](https://github.com/WebKit/webkit/commit/2339ace9ed48b4fe3377be84cc0b61c0e6966f72)

**Test References**
- [prune-statistics](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/prune-statistics.html)

### ITP Debug mode

ITP comes with a developer debug setting that can be enabled in *Develop -> Experimental Features -> ITP Debug Mode*. This feature will:
- Show log messages for ITP classifications and storage decisions
- Set `http://3rdpartytestwebkit.org` to be prevalent
- Allow developers to set their own custom domain as prevalent

Read more about working with ITP Debug mode in [the Safari Technology Preview 62 release notes](https://webkit.org/blog/8387/itp-debug-mode-in-safari-technology-preview-62/).

**Source References**
- [#187835](https://bugs.webkit.org/show_bug.cgi?id=187835) - [Enable basic functionality in experimental debug mode](https://github.com/WebKit/webkit/commit/2ba7c028d65df9f15f4a662ec430c4a1b54d04ac)
- [#187918](https://bugs.webkit.org/show_bug.cgi?id=187918) - [Add logging of Storage Access API use in experimental debug mode](https://github.com/WebKit/webkit/commit/9f4086f92140342d44184fb558be6b449d5d88bf)

**Test References**
- [enable-debug-mode](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/enable-debug-mode.html)
- [set-custom-prevalent-resource-in-debug-mode](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/set-custom-prevalent-resource-in-debug-mode.html)

### Telemetry

ITP statistics and classification are mostly on device, but there is some telemetry that is sent to Apple. This is limited to an aggregated set of metrics for things like the number of prevalent resources that had user interaction, the median number of times a top prevalent domain has been loaded as a subframe, etc.

**Source References**
- [WebResourceLoadStatisticsTelemetry.cpp](https://github.com/WebKit/webkit/blob/master/Source/WebKit/UIProcess/WebResourceLoadStatisticsTelemetry.cpp)
- [#173499](https://bugs.webkit.org/show_bug.cgi?id=173499) - [Add telemetry](https://github.com/WebKit/webkit/commit/a603d9eb7d9d0e839dd0826101987c474180c3cd)

**Test References**

- [telemetry-generation](https://github.com/WebKit/webkit/blob/master/LayoutTests/http/tests/resourceLoadStatistics/telemetry-generation.html)
