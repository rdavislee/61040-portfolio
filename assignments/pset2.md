# URL Shortening Service - Problem Set Answers

## Concept Questions

### 1. Contexts
Contexts are so that the URL shortener can have multiple sets of short URLs, where within those sets the short URLs are unique with respect to each other but not necessarily across all contexts. This allows us to associate contexts with shortUrlBases, where the shortUrlSuffixes for a single shortUrlBase are unique, but if the shortUrlBase differs the shortUrlSuffix can be the same.

### 2. Storing Used Strings
The NonceGeneration must store used strings to guarantee uniqueness. The set of used strings in the specification is related to the counter in the implementation in that the length of the used set must always equal the counter in order to ensure uniqueness. In other words, AF(used set) = the strings that have been used in a given context where the length of the used set is always equal to the counter in the implementation.

### 3. Words as Nonces
An advantage is that a word is far easier to enter and remember than a random sequence of characters. Some disadvantages could be that words can result in longer URLS and that a randomly generated word could be inappropriate. I would modify NonceGeneration by adding a word set to each context that is by default every dictionary word but can be initialized as any set of words. Then, generate would randomly select a word from the words set, use it as the shortURL, add it to the used set, and remove it from the words set.

## Synchronization Questions

### 1. Partial Matching
The generate sync only matches on shortUrlBase because it is only generating a random short url; it does not need a target url yet. The register sync needs both because it is actually creating the url mapping. It is waiting for both the user request and the nonce generation to complete before registering.

### 2. Omitting Names
This convention cannot be used if the argument names do not match between the source and destination. For example, the NonceGeneration generates a "nonce" and the register takes a "shortUrlSuffix," so one of these needs to use a colon to distinguish.

### 3. Inclusion of Request
The first two syncs include requests because they are triggered by external user actions while the third sync is triggered internally by the completion of a register action.

### 4. Fixed Domain - Following Changes

**Generate**
```
When Request.shrtenUrl()
Then NonceGeneration.generate(context: *fixed domain*)
```

**Register**
```
When
  Request.shortenUrl (targetUrl)
  NonceGeneration.generate (): (nonce)
Then
  UrlShortening.register ( shortUrlSuffix: nonce, shortUrlBase: *fixed domain*, targetUrl)
```

setExpiry stays the same.

### 5. Adding a Sync

**Sync expire**
```
When ExpiringResource.expireResource(): (resource: shortUrl)
Then UrlShortening.delete(shortUrl)
```

## Extending the Design

### New Concepts

#### concept Analytics [Resource]
```
purpose track access counts for resources
principle after incrementing count for a resource, getCount returns the updated total
state
  a set of Resources with
    a count Number (default 0)
actions
  increment (resource: Resource)
    effect increments the count for resource by 1
  getCount (resource: Resource, user: User, allowed: Boolean): (count: Number)
    requires resource exists in state
    shows current count to User if allowed is True
  resetCount (resource: Resource)
    requires resource exists in state
    effect sets count to 0 for resource
```

#### concept AccessControl [Resource, User]
```
purpose restrict access to resources based on ownership
principle after assigning owner to resource, only that user can access it
state
  a set of Resources with
    an owner User
actions
  setOwner (resource: Resource, user: User)
    requires no owner exists for resource
    effect assigns user as owner of resource
  checkAccess (resource: Resource, user: User): (allowed: Boolean)
    effect returns true if user is owner of resource, false otherwise
  removeOwner (resource: Resource)
    requires owner exists for resource
    effect removes ownership of resource
```

### Essential Synchronizations

#### sync trackOwnership
```
when
  UrlShortening.register():(shortUrl)
  Request.shortenUrl(user)
then
  AccessControl.setOwner(resource: shortUrl, user)
  Analytics.resetCount(resource: shortUrl)
```

#### sync trackLookup
```
when UrlShortening.lookup (shortUrl)
then Analytics.increment (resource: shortUrl)
```

#### sync requestAnalytics
```
when Request.getAnalytics (shortUrl, user)
then
  AccessControl.checkAccess (resource: shortUrl, user): (allowed)
  Analytics.getCount (resource: shortUrl, user, allowed): ()
```

## Modularity Assessment

### a) Allowing users to choose their own short URLs

**Implementation:** Add a new sync that bypasses NonceGeneration when user provides a custom suffix:

```
sync customRegister
  when Request.customShortenUrl (customSuffix, shortUrlBase, targetUrl, user)
  then 
    UrlShortening.register (shortUrlSuffix: customSuffix, shortUrlBase, targetUrl)
    AccessControl.setOwner (resource: shortUrlBase/customSuffix, user)
```

**Assessment:** Highly modular - no changes to existing concepts needed.

### b) Using "word as nonce" strategy

**Implementation:** Modify NonceGeneration concept to support word lists (as discussed earlier), or create a new WordNonceGeneration concept and use it in place of NonceGeneration in the synchronizations. 

**Assessment:** Modular - can swap nonce generation strategies without affecting other concepts.

### c) Including target URL in analytics

**Implementation:** Extend Analytics concept to support grouping by an additional key:

```
concept Analytics [Resource, GroupKey]
  state: resources with count and groupKey
  actions: add getCountByGroup(groupKey): returns total across resources
```

Then sync would pass targetUrl as the groupKey.

**Assessment:** Reasonably modular - only Analytics concept needs enhancement.

### d) Generate short URLs that are not easily guessed

**Implementation:** Create a new SecureNonceGeneration concept that uses cryptographic randomness:

```
concept SecureNonceGeneration [Context]
  // Uses cryptographically secure random generation
  // Same interface as NonceGeneration but different implementation
```

**Assessment:** Perfectly modular - just swap the nonce generation concept.

### e) Supporting analytics for non-registered users

**Assessment:** Should not be included. This feature would violate privacy by exposing analytics to non-creators, conflict with the AccessControl concept's purpose, create potential security risks (competitors could track campaign performance), and contradict the requirement that "analytics should not be public"

If needed, you'd have to fundamentally redesign AccessControl to support different permission levels, but this would compromise the security model of the system.