# Problem Set 1

## Exercise 1

### Question 1
**Invariant 1:** Count for an item in a registry can never be less than 0

**Invariant 2:** Every purchase in a registry must have a corresponding request in the registry that it applies to

The more important is Invariant 1 because a negative value for count would be nonsensical and potentially code-breaking. This invariant is kept by using a precondition on new purchases that the count of the purchase is equal to or less than the count of the request.

### Question 2
Invariant 1 could be broken when adding an item because there is no constraint on the initial amount of count, allowing for an initial value below 0. This is also true for purchase, allowing for nonsensical negative purchases. This could be fixed by adding a precondition on `addItem` and `purchase` to have count greater than 0.

### Question 3
The open and close mechanic allows for a registry to be opened and closed repeatedly. This mechanic should be allowed, in the case of an event such as a wedding getting delayed and the hosts wanting to reopen the gift page.

### Question 4
This would not matter in practice; once closed and never reopened, it would be essentially the same as deleting it.

### Question 5
- The owner may want to see what has been purchased from the gift list in a query
- A gift giver may want to see what hasn't been purchased in a query

### Question 6
Add a flag to the state for each registry called `visible`. When `registry.visible = true` and `registry.active = true`, no query by the owner can access counts of items.

### Question 7
Something like SKU codes would be preferable because they are proven to be unique identifiers for each item and can be used as keys to find the rest of the information. This protects against duplicate items.

## Exercise 2

### Question 1
A set of users with a string username and a string password.

### Question 2

**register(username: String, password: String): (user: User)**
- **Requires:** No registered user `u` have `u.username = username`
- **Effects:**
  - Creates a new user `u` with `u.username = username` and `u.password = password`
  - `users += u`
  - Returns `u`

**authenticate(username: String, password: String): (user: User)**
- **Requires:** For some `u` in users, `u.username = username` and `u.password = password`
- **Effects:** Returns `u`

### Question 3
**Invariant:** Each username is associated with at most one user.

It is preserved because the register action's precondition checks that no existing user has the same username, and authenticate does not modify state.

### Question 4

#### Concept: PasswordAuthenticationEmail

**Purpose:** Limit access to known users with email verification

**Principle:** After a user registers with username and password, they receive a token to confirm, and after confirmation they can authenticate and be treated each time as the same user

**State:**
A set of users with:
- A username `String`
- A password `String`
- A token `String`
- A confirmed `Boolean`

**Actions:**

**register(username: String, password: String): (user: User, token: String)**
- **Requires:** No registered user `u` have `u.username = username`
- **Effects:**
  - Creates a secret token String `t`
  - Creates a new user `u` with `u.username = username`, `u.password = password`, `u.token = t`, `u.confirmed = False`
  - `users += u`
  - Returns `u`

**confirm(username: String, token: String): ()**
- **Requires:** For some `u` in users, `u.username = username`, `u.token = token`, `u.confirmed = False`
- **Effects:** `u.confirmed = True`

**authenticate(username: String, password: String): (user: User)**
- **Requires:** For some `u` in users, `u.username = username`, `u.password = password`, and `u.confirmed = True`
- **Effects:** Returns `u`

## Exercise 3

### Concept: PersonalAccessToken

**Purpose:** Provide revocable scoped credentials for account access

**Principle:** User creates one or more named tokens with specific scopes and optional expiration; each token can authenticate API/CLI actions in their scopes; tokens can be individually revoked

**State:**
A set of Tokens with:
- An owner `User`
- A name `String`
- A tokenString `String`
- A set of scopes `Scope`
- An optional expiration `Date`
- An active `Boolean`

**Actions:**

**createToken(owner: User, name: String, scopes: Set\<Scope\>, expiration: Date?): (tokenString: String)**
- **Requires:** Owner is authenticated via password
- **Effects:**
  - Create new token `t` for owner with `t.name = name`, `t.scopes = scopes`, and `t.expiration = expiration`
  - Generate unique tokenString `s`
  - `t.tokenString = s`
  - Set active to true for the token
  - Return tokenString

**authenticate(tokenString: String): (user: User, scopes: Set\<Scope\>)**
- **Requires:** Token exists with this tokenString, is active and not expired
- **Effects:** Return the owner and scopes of the matching token

**revokeToken(owner: User, token: Token): ()**
- **Requires:** Token exists and owner owns it
- **Effects:** Set token's active to false

### Differences from PasswordAuthentication
- Users can create many personal access tokens (PATs) with different names/purposes
- PATs have limited scopes
- PATs have independent life cycles

### How to Change GitHub Documentation
- Clarify conceptual differences upfront by starting with a statement about the differences between PATs and a password/username
- Don't say "Personal access tokens are like passwords" (see the Keeping your personal access tokens secure page), it adds to the confusion
- Add a comparison table between PATs and passwords

## Exercise 4

### Conference Room Booking

#### Concept: ConferenceRoomBooking

**Purpose:** Prevent conflicts in room usage

**Principle:** Users can book available rooms for specific time periods; a room cannot be double-booked; users can cancel their own bookings

**State:**
A set of Bookings with:
- A booker `User`
- A room `Room`
- A startTime `DateTime`
- An endTime `DateTime`
- An optional description `String`

**Actions:**

**book(user: User, room: Room, start: DateTime, end: DateTime, desc: String?): (booking: Booking)**
- **Requires:**
  - `start < end`
  - No existing booking for room overlaps `[start, end)`
- **Effects:**
  - Create new booking `b` with `b.booker = user`, `b.room = room`
  - Set `b.startTime = start`, `b.endTime = end`
  - Set `b.description = desc` if provided
  - Add `b` to Bookings
  - Return booking `b`

**cancel(user: User, booking: Booking): ()**
- **Requires:**
  - Booking exists in Bookings
  - `booking.booker = user`
- **Effects:** Remove booking from Bookings

**Notes:** I chose continuous time intervals rather than discrete slots for maximum flexibility. The overlap check prevents any double-booking. I included optional descriptions for meeting purposes.

### Address Verification

#### Concept: AddressVerification

**Purpose:** Verify user knowledge of registered address

**Principle:** An authority stores trusted address records for users; verification requests challenge users to provide address components; matching provided components against stored records confirms address knowledge

**State:**
A set of AddressRecords with:
- A user `User`
- An authority `Authority`
- A streetAddress `String`
- A city `String`
- A state `String`
- A zipCode `String`
- A country `String`

**Actions:**

**registerAddress(authority: Authority, user: User, street: String, city: String, state: String, zip: String, country: String)**
- **Requires:** Authority is trusted source
- **Effects:**
  - Create or update address record `a` for user
  - Set `a.user = user`, `a.authority = authority`
  - Set `a.streetAddress = street`, `a.city = city`
  - Set `a.state = state`, `a.zipCode = zip`, `a.country = country`
  - Add or update `a` in AddressRecords

**verifyFull(user: User, street: String, city: String, state: String, zip: String): (verified: Boolean)**
- **Requires:** Address record exists for user in AddressRecords
- **Effects:**
  - Find address record `a` for user
  - Check if `a.streetAddress = street` and `a.city = city`
  - Check if `a.state = state` and `a.zipCode = zip`
  - Return true if all match, false otherwise

**verifyPartial(user: User, zip: String): (verified: Boolean)**
- **Requires:** Address record exists for user in AddressRecords
- **Effects:**
  - Find address record `a` for user
  - Return true if `a.zipCode = zip`
  - Return false otherwise

**Notes:** This concept is inherently distributed—the authority maintains the trusted address, while verification happens at the point of use. I included both full and partial verification to handle different security requirements.

### Time-based OTP

#### Concept: TimeBasedOTP

**Purpose:** Provide possession-based second authentication factor

**Principle:** User and server share a secret key; both generate time-synchronized tokens using the secret; user proves possession of authenticator by providing current token; tokens expire after short time window

**State:**
A set of Authenticators with:
- A user `User`
- A secretKey `String`
- A timeStep `Integer`
- A windowSize `Integer`
- A lastUsedCounter `Integer`

**Actions:**

**enroll(user: User): (secret: String, qrCode: String)**
- **Requires:**
  - User is authenticated via primary method
  - No authenticator exists for user
- **Effects:**
  - Generate new secret key `s`
  - Create authenticator `a` with `a.user = user`
  - Set `a.secretKey = s`, `a.timeStep = 30`, `a.windowSize = 1`
  - Set `a.lastUsedCounter = 0`
  - Generate QR code `q` encoding secret
  - Add `a` to Authenticators
  - Return secret `s` and QR code `q`

**verify(user: User, token: String): (valid: Boolean)**
- **Requires:** Authenticator `a` exists for user in Authenticators
- **Effects:**
  - Find authenticator `a` for user
  - Calculate current counter `c = floor(currentTime/a.timeStep)`
  - For each counter `i` in `[c-a.windowSize, c+a.windowSize]`:
    - Generate expected token `t = HMAC(a.secretKey, i)`
    - If token matches `t` and `i > a.lastUsedCounter`:
      - Set `a.lastUsedCounter = i`
      - Return true
  - Return false if no valid match found

**disable(user: User)**
- **Requires:**
  - User is authenticated via primary method
  - Authenticator exists for user
- **Effects:** Remove authenticator for user from Authenticators

**Notes:** This concept improves security by requiring the user to use something they have and something they know in conjunction. It can prevent password-only attacks. It remains vulnerable to real-time phishing attacks.