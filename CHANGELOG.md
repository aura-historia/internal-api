# Changelog

All notable changes to the Aura-Historia Backend API will be documented in this file.

This changelog is for internal communication between frontend and backend teams. It documents what is new, what has changed, and what has been removed for each API update.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## 2025-11-07 - API Personalization and User Resource Path Restructuring

This update introduces personalized API responses for item endpoints and reorganizes user-specific resources under a `/api/v1/me/` prefix for better API structure and clarity.

### Added

#### Personalized Item Responses

Item retrieval and search endpoints now support optional authentication, returning personalized data that includes user-specific state when a valid JWT token is provided.

**New Schema: `PersonalizedGetItemData`**
- Wrapper for item data with optional user-specific state
- Properties:
  - `item` (GetItemData, required): Complete item information
  - `userState` (ItemUserStateData, optional): User-specific state (only present when authenticated)

**Example Response (Authenticated User)**:
```json
{
  "item": {
    "itemId": "550e8400-e29b-41d4-a716-446655440000",
    "eventId": "550e8400-e29b-41d4-a716-446655440001",
    "shopId": "550e8400-e29b-41d4-a716-446655440000",
    "shopsItemId": "6ba7b810",
    "shopName": "My Shop",
    "title": {
      "text": "Amazing Product",
      "language": "en"
    },
    "state": "AVAILABLE",
    "url": "https://my-shop.com/products/amazing-product",
    "created": "2024-01-01T10:00:00Z",
    "updated": "2024-01-01T12:00:00Z"
  },
  "userState": {
    "watchlist": {
      "watching": true,
      "notifications": false
    }
  }
}
```

**Example Response (Anonymous User)**:
```json
{
  "item": {
    "itemId": "550e8400-e29b-41d4-a716-446655440000",
    "eventId": "550e8400-e29b-41d4-a716-446655440001",
    "shopId": "550e8400-e29b-41d4-a716-446655440000",
    "shopsItemId": "6ba7b810",
    "shopName": "My Shop",
    "title": {
      "text": "Amazing Product",
      "language": "en"
    },
    "state": "AVAILABLE",
    "url": "https://my-shop.com/products/amazing-product",
    "created": "2024-01-01T10:00:00Z",
    "updated": "2024-01-01T12:00:00Z"
  }
}
```

**New Schema: `PersonalizedItemSearchResultData`**
- Paginated collection of personalized items using cursor-based pagination
- Properties:
  - `items` (array of PersonalizedGetItemData, required): Personalized items in current page
  - `size` (integer, required): Number of items in current page
  - `total` (integer, optional): Total count of matching items
  - `searchAfter` (array, optional): Cursor for next page

**New Schema: `ItemUserStateData`**
- User-specific state information for an item
- Properties:
  - `watchlist` (WatchlistUserStateData, required): Watchlist-related state

**New Schema: `WatchlistUserStateData`**
- Watchlist-specific user state for an item
- Properties:
  - `watching` (boolean, required): Whether the item is on the user's watchlist
  - `notifications` (boolean, required): Whether notifications are enabled for this watchlist item

### Changed

#### GET /api/v1/items/{shopId}/{shopsItemId}

This endpoint now supports optional authentication and returns personalized data.

**Authentication Changed**:
- **Before**: No authentication support
- **After**: Optional `Authorization` header (Cognito JWT Bearer token)

**Response Schema Changed**:
- **Before**: Returns `GetItemData` directly
- **After**: Returns `PersonalizedGetItemData` (wrapper with `item` and optional `userState`)

**New Request Header**:
- `Authorization` (optional): Cognito JWT token
  - Format: `Bearer [JWT token]`
  - When provided: Response includes `userState` with watchlist information
  - When omitted: Response contains only `item` without `userState`

**Response Body Structure**:
```json
{
  "item": { /* GetItemData */ },
  "userState": { /* ItemUserStateData - only when authenticated */ }
}
```

**Migration Impact**:
- Frontend must update response parsing to access item data via `response.item` instead of directly
- User state is available via `response.userState` when user is authenticated
- Anonymous requests work identically but with wrapped response structure

#### POST /api/v1/items/search

This endpoint now supports optional authentication and returns personalized data for each item in search results.

**Authentication Changed**:
- **Before**: No authentication support
- **After**: Optional `Authorization` header (Cognito JWT Bearer token)

**Response Schema Changed**:
- **Before**: Returns `ItemSearchResultData` with array of `GetItemData`
- **After**: Returns `PersonalizedItemSearchResultData` with array of `PersonalizedGetItemData`

**New Request Header**:
- `Authorization` (optional): Cognito JWT token
  - Format: `Bearer [JWT token]`
  - When provided: Each item in results includes `userState` with watchlist information
  - When omitted: Items contain only item data without `userState`

**Response Body Structure**:
```json
{
  "items": [
    {
      "item": { /* GetItemData */ },
      "userState": { /* ItemUserStateData - only when authenticated */ }
    }
  ],
  "size": 21,
  "total": 84,
  "searchAfter": "[2999, \"6ba7b810-9dad-11d1-80b4-00c04fd430c8\"]"
}
```

**Migration Impact**:
- Frontend must update item access from `response.items[i]` to `response.items[i].item`
- User state per item available via `response.items[i].userState` when authenticated
- Pagination fields (`size`, `total`, `searchAfter`) remain at root level

#### User Resource Path Restructuring

All user-specific endpoints have been reorganized under the `/api/v1/me/` prefix to better reflect their nature as user-scoped resources.

**Watchlist Endpoints**:
- `GET /api/v1/watchlist` → `GET /api/v1/me/watchlist`
- `POST /api/v1/watchlist` → `POST /api/v1/me/watchlist`
- `DELETE /api/v1/watchlist/{shopId}/{shopsItemId}` → `DELETE /api/v1/me/watchlist/{shopId}/{shopsItemId}`
- `PATCH /api/v1/watchlist/{shopId}/{shopsItemId}` → `PATCH /api/v1/me/watchlist/{shopId}/{shopsItemId}`

**Search Filter Endpoints**:
- `GET /api/v1/search-filters` → `GET /api/v1/me/search-filters`
- `POST /api/v1/search-filters` → `POST /api/v1/me/search-filters`
- `GET /api/v1/search-filters/{searchFilterId}` → `GET /api/v1/me/search-filters/{searchFilterId}`
- `PATCH /api/v1/search-filters/{searchFilterId}` → `PATCH /api/v1/me/search-filters/{searchFilterId}`
- `DELETE /api/v1/search-filters/{searchFilterId}` → `DELETE /api/v1/me/search-filters/{searchFilterId}`

**Migration Impact**:
- All API clients must update endpoint URLs to use `/api/v1/me/` prefix for user resources
- Request and response structures remain unchanged - only the path has changed
- Authentication requirements remain unchanged (all still require JWT)
- Location headers in POST responses now point to new paths

**Example URL Changes**:
- Old: `https://api.aura-historia.com/api/v1/watchlist`
- New: `https://api.aura-historia.com/api/v1/me/watchlist`

- Old: `https://api.aura-historia.com/api/v1/search-filters/6ba7b810-9dad-11d1-80b4-00c04fd430c8`
- New: `https://api.aura-historia.com/api/v1/me/search-filters/6ba7b810-9dad-11d1-80b4-00c04fd430c8`

### Migration Guide

For frontend developers integrating these changes:

1. **Update Item GET Endpoint**:
   ```typescript
   // Before
   const item: GetItemData = await getItem(shopId, shopsItemId);
   
   // After - Anonymous
   const response: PersonalizedGetItemData = await getItem(shopId, shopsItemId);
   const item = response.item;
   
   // After - Authenticated
   const response: PersonalizedGetItemData = await getItem(shopId, shopsItemId, token);
   const item = response.item;
   const isWatching = response.userState?.watchlist.watching ?? false;
   const notificationsEnabled = response.userState?.watchlist.notifications ?? false;
   ```

2. **Update Complex Search Endpoint**:
   ```typescript
   // Before
   const result: ItemSearchResultData = await searchItems(filter);
   const items: GetItemData[] = result.items;
   
   // After - Anonymous
   const result: PersonalizedItemSearchResultData = await searchItems(filter);
   const items = result.items.map(p => p.item);
   
   // After - Authenticated
   const result: PersonalizedItemSearchResultData = await searchItems(filter, token);
   const itemsWithState = result.items.map(personalized => ({
     item: personalized.item,
     watching: personalized.userState?.watchlist.watching ?? false,
     notifications: personalized.userState?.watchlist.notifications ?? false
   }));
   ```

3. **Update Watchlist Endpoint URLs**:
   ```typescript
   // Before
   const baseUrl = 'https://api.aura-historia.com/api/v1/watchlist';
   
   // After
   const baseUrl = 'https://api.aura-historia.com/api/v1/me/watchlist';
   ```

4. **Update Search Filter Endpoint URLs**:
   ```typescript
   // Before
   const baseUrl = 'https://api.aura-historia.com/api/v1/search-filters';
   
   // After
   const baseUrl = 'https://api.aura-historia.com/api/v1/me/search-filters';
   ```

5. **Handle Optional Authentication**:
   - Item endpoints can now be called without authentication
   - When calling without auth, expect `userState` to be absent in responses
   - Use optional chaining when accessing user state: `response.userState?.watchlist.watching`

### Backend Implementation Details

- Authentication is verified via Cognito access token
- User ID is extracted from JWT when present
- Watchlist state is queried from DynamoDB when user is authenticated
- Anonymous requests skip user state lookup for better performance
- Response structure uses a generic `Personalized<Item, UserState>` wrapper
- Path changes implemented via AWS API Gateway route updates
- All endpoints maintain backward compatibility for request parameters and bodies

### Removed

No endpoints or schemas have been removed in this update. The old watchlist and search-filter paths at `/api/v1/watchlist*` and `/api/v1/search-filters*` are deprecated and will be removed in a future update. Clients should migrate to the new `/api/v1/me/*` paths immediately.

## 2025-10-24 - Search Filter Names and Shop Search Keyset Pagination

This update introduces user-defined names for search filters and migrates shop search from offset-based pagination to cursor-based (keyset) pagination for better performance and scalability.

### Added

#### Search Filter Names

All search filter endpoints now support a `searchFilterName` field that allows users to assign custom names to their saved search filters.

**New Field in `UserSearchFilterData`:**
```json
{
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "searchFilterId": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
  "searchFilterName": "My Tech Store Search",
  "searchFilter": {
    "language": "en",
    "currency": "USD",
    "itemQuery": "smartphone"
  },
  "created": "2024-01-01T10:00:00Z",
  "updated": "2024-01-01T12:00:00Z"
}
```

**New Schema: `PostUserSearchFilterData`**
- Used for creating search filters
- Requires both `searchFilterName` (string, max 255 characters) and `searchFilter` object

**New Schema: `PatchUserSearchFilterData`**
- Used for updating search filters
- Optional `searchFilterName` field for updating the filter name
- Optional `searchFilter` object (type: `PatchSearchFilterData`) for updating filter criteria
- Both fields are optional - either or both can be updated

**New Schema: `PatchSearchFilterData`**
- Contains the actual search filter criteria that can be partially updated
- All fields from `SearchFilterData` are optional in this schema

### Changed

#### POST /api/v1/search-filters

**Request Body Structure Changed:**

Before:
```json
{
  "language": "en",
  "currency": "USD",
  "itemQuery": "smartphone",
  "shopNameQuery": "Tech Store",
  "price": {
    "min": 1000,
    "max": 5000
  }
}
```

After:
```json
{
  "searchFilterName": "My Tech Store Search",
  "searchFilter": {
    "language": "en",
    "currency": "USD",
    "itemQuery": "smartphone",
    "shopNameQuery": "Tech Store",
    "price": {
      "min": 1000,
      "max": 5000
    }
  }
}
```

**Response:** Now includes `searchFilterName` field in the returned `UserSearchFilterData`.

#### PATCH /api/v1/search-filters/{searchFilterId}

**Request Body Structure Changed:**

The PATCH endpoint now supports updating the search filter name separately from the filter criteria. The request body structure has been reorganized to nest the filter criteria.

Before (flat structure):
```json
{
  "shopNameQuery": "Updated Store",
  "price": {
    "min": 2000,
    "max": 8000
  }
}
```

After (nested structure):
```json
{
  "searchFilterName": "Updated Filter Name",
  "searchFilter": {
    "shopNameQuery": "Updated Store",
    "price": {
      "min": 2000,
      "max": 8000
    }
  }
}
```

**Update Options:**
- Update name only: `{ "searchFilterName": "New Name" }`
- Update filter criteria only: `{ "searchFilter": { "language": "de" } }`
- Update both: `{ "searchFilterName": "New Name", "searchFilter": { "language": "de" } }`
- No update (returns existing): `{}`

**Response:** Now includes `searchFilterName` field in the returned `UserSearchFilterData`.

#### GET /api/v1/search-filters/{searchFilterId}

**Response:** Now includes `searchFilterName` field in the returned `UserSearchFilterData`.

#### GET /api/v1/search-filters

**Response:** All items in the `items` array now include `searchFilterName` field.

#### POST /api/v1/shops/search

**Pagination Changed from Offset-Based to Cursor-Based (Keyset Pagination):**

Query Parameters:
- **Removed:** `from` (offset) parameter
- **Added:** `searchAfter` parameter (JSON array cursor)

**`searchAfter` Parameter:**
- Type: JSON array
- Description: Cursor value from previous response's `searchAfter` field
- Format: `[sortValue, shopId]` (heterogeneous array)
- Example: `["Tech Store Premium", "550e8400-e29b-41d4-a716-446655440000"]`
- Usage: Pass the `searchAfter` value from the previous page to get the next page

**Response Schema Changed:**

Before (`ShopSearchResultData`):
```json
{
  "items": [...],
  "from": 0,
  "size": 21,
  "total": 127
}
```

After (`ShopSearchResultData`):
```json
{
  "items": [...],
  "size": 2,
  "searchAfter": "[\"Tech Store Basic\", \"660f9500-f39c-42e5-b827-556766550001\"]",
  "total": 42
}
```

**Response Fields:**
- `items`: Array of shop results (unchanged)
- `size`: Number of items in current page (unchanged, now required)
- `searchAfter`: Cursor for next page (nullable, present when more results available)
- `total`: Total number of matching items (unchanged, nullable)
- **Removed:** `from` field

**Pagination Workflow:**
1. First request: Omit `searchAfter` parameter
2. Check response for `searchAfter` field
3. If `searchAfter` is present, use it in next request to get the next page
4. Repeat until `searchAfter` is not present in response

**Benefits:**
- Better performance for large result sets
- Consistent pagination even when data changes
- No risk of duplicates or skipped items during pagination

### Removed

#### Schema: `SearchFilterDataPatch`

This schema has been replaced by `PatchUserSearchFilterData` and `PatchSearchFilterData` to support the new nested structure for partial updates.

## 2025-10-21 - Item Event History Enhanced with Old and New Values

This update enhances the item history events to include both old and new values for state and price changes, making it easier for the frontend to display what changed without computing differences. This provides richer context for tracking item history and enables better UI experiences when showing price drops, increases, and state transitions.

### Changed

#### GET /api/v1/items/{shopId}/{shopsItemId}?history=true

The history array in the response now includes more detailed information for each event type. Event payloads have been restructured to provide both old and new values for changes.

**Breaking Changes to Event Payload Structure:**

1. **State Change Events** (STATE_LISTED, STATE_AVAILABLE, STATE_RESERVED, STATE_SOLD, STATE_REMOVED, STATE_UNKNOWN)
   
   **Before:**
   ```json
   {
     "eventType": "STATE_AVAILABLE",
     "payload": "AVAILABLE"
   }
   ```
   
   **After:**
   ```json
   {
     "eventType": "STATE_AVAILABLE",
     "payload": {
       "oldState": "LISTED",
       "newState": "AVAILABLE"
     }
   }
   ```

2. **Price Discovered Events** (PRICE_DISCOVERED)
   
   **Before:**
   ```json
   {
     "eventType": "PRICE_DISCOVERED",
     "payload": {
       "currency": "EUR",
       "amount": 2999
     }
   }
   ```
   
   **After:**
   ```json
   {
     "eventType": "PRICE_DISCOVERED",
     "payload": {
       "newPrice": {
         "currency": "EUR",
         "amount": 2999
       }
     }
   }
   ```

3. **Price Change Events** (PRICE_DROPPED, PRICE_INCREASED)
   
   **Before:**
   ```json
   {
     "eventType": "PRICE_DROPPED",
     "payload": {
       "currency": "EUR",
       "amount": 2499
     }
   }
   ```
   
   **After:**
   ```json
   {
     "eventType": "PRICE_DROPPED",
     "payload": {
       "oldPrice": {
         "currency": "EUR",
         "amount": 2999
       },
       "newPrice": {
         "currency": "EUR",
         "amount": 2499
       }
     }
   }
   ```

4. **Price Removed Events** (PRICE_REMOVED)
   
   **Before:**
   ```json
   {
     "eventType": "PRICE_REMOVED",
     "payload": null
   }
   ```
   
   **After:**
   ```json
   {
     "eventType": "PRICE_REMOVED",
     "payload": {
       "oldPrice": {
         "currency": "EUR",
         "amount": 2999
       }
     }
   }
   ```

#### New Data Types

**ItemEventStateChangedPayloadData**
- Used for all state change events
- Properties:
  - `oldState` (ItemStateData, required): The previous state of the item
  - `newState` (ItemStateData, required): The new state of the item

**ItemEventPriceDiscoveredPayloadData**
- Used for PRICE_DISCOVERED events when a price is first detected
- Properties:
  - `newPrice` (PriceData, required): The newly discovered price

**ItemEventPriceChangedPayloadData**
- Used for PRICE_DROPPED and PRICE_INCREASED events
- Properties:
  - `oldPrice` (PriceData, required): The previous price
  - `newPrice` (PriceData, required): The new price

**ItemEventPriceRemovedPayloadData**
- Used for PRICE_REMOVED events when a price is removed from an item
- Properties:
  - `oldPrice` (PriceData, required): The price that was removed

### Migration Guide

For frontend developers integrating these changes:

1. **Update item history event handling**:
   - All state change events now return an object with `oldState` and `newState` fields
   - Access state using `event.payload.newState` instead of just `event.payload`

2. **Update price event handling**:
   - PRICE_DISCOVERED events now have the price under `event.payload.newPrice`
   - PRICE_DROPPED and PRICE_INCREASED events now provide both `oldPrice` and `newPrice`
   - PRICE_REMOVED events now include the `oldPrice` that was removed

3. **Display improvements**:
   - You can now show "Changed from X to Y" messages for state transitions
   - Display price drops/increases with the actual old and new values
   - Show what price was removed instead of just indicating removal

4. **Example complete event object**:
   ```json
   {
     "eventType": "PRICE_DROPPED",
     "itemId": "550e8400-e29b-41d4-a716-446655440000",
     "eventId": "550e8400-e29b-41d4-a716-446655440001",
     "shopId": "550e8400-e29b-41d4-a716-446655440000",
     "shopsItemId": "6ba7b810",
     "payload": {
       "oldPrice": {
         "currency": "USD",
         "amount": 3499
       },
       "newPrice": {
         "currency": "USD",
         "amount": 2999
       }
     },
     "timestamp": "2024-01-15T14:30:00Z"
   }
   ```

### Backend Implementation Details

- Event payloads are now separate types: `ItemEventStateChangedPayloadData`, `ItemEventPriceDiscoveredPayloadData`, `ItemEventPriceChangedPayloadData`, and `ItemEventPriceRemovedPayloadData`
- Price discovery (first time a price is detected) is now distinguished from price changes with separate payload structure
- All state and price change events capture the old value before modification occurs
- The `ItemEventPayloadData` enum uses `oneOf` discriminator to allow different payload structures per event type

## 2025-10-14 - Watchlist Item Identification Refactoring

This update improves the watchlist item API by simplifying item identification. Watchlist items are now uniquely identified by their composite key (shopId + shopsItemId) rather than requiring the creation timestamp. This makes the API more intuitive and reduces the likelihood of errors.

### Changed

#### DELETE /api/v1/watchlist/{shopId}/{shopsItemId}

The DELETE endpoint has been simplified to remove the `created` query parameter requirement:

**Before**:
```
DELETE /api/v1/watchlist/{shopId}/{shopsItemId}?created={timestamp}
```

**After**:
```
DELETE /api/v1/watchlist/{shopId}/{shopsItemId}
```

**Changes**:
- **Removed** `created` query parameter - No longer required to identify the watchlist entry
- Watchlist items are now uniquely identified by the combination of `shopId` and `shopsItemId` path parameters
- Simplified error handling - No more `BAD_QUERY_PARAMETER_VALUE` or `INVALID_RFC3339_TIMESTAMP` errors related to the `created` parameter

**Migration Impact**:
- Frontend applications must update DELETE requests to remove the `created` query parameter
- The endpoint now relies solely on path parameters `{shopId}/{shopsItemId}` for item identification
- Error handling code for missing/invalid `created` timestamps can be removed

#### PATCH /api/v1/watchlist/{shopId}/{shopsItemId}

The PATCH endpoint has been simplified to remove the `created` query parameter requirement:

**Before**:
```
PATCH /api/v1/watchlist/{shopId}/{shopsItemId}?created={timestamp}
```

**After**:
```
PATCH /api/v1/watchlist/{shopId}/{shopsItemId}
```

**Changes**:
- **Removed** `created` query parameter - No longer required to identify the watchlist entry
- Watchlist items are now uniquely identified by the combination of `shopId` and `shopsItemId` path parameters
- Simplified error handling - No more `BAD_QUERY_PARAMETER_VALUE` or `INVALID_RFC3339_TIMESTAMP` errors related to the `created` parameter
- Request body and response structure remain unchanged

**Migration Impact**:
- Frontend applications must update PATCH requests to remove the `created` query parameter
- The endpoint now relies solely on path parameters `{shopId}/{shopsItemId}` for item identification
- Error handling code for missing/invalid `created` timestamps can be removed

#### POST /api/v1/watchlist

The POST endpoint now returns the created watchlist item data in the response body:

**Changes**:
- **Added** response body containing `WatchlistItemPatchResponse` data
- Response now includes all watchlist item metadata: `shopId`, `shopsItemId`, `itemId`, `notifications`, `created`, and `updated`
- The `Location` header is still included pointing to the created resource
- Default `notifications` value is `false` for newly created items
- Both `created` and `updated` timestamps are set to the same value when an item is first added

**Response Body Schema**: `WatchlistItemPatchResponse`
- `shopId` (string, uuid, required): Shop identifier
- `shopsItemId` (string, required): Shop's item identifier  
- `itemId` (string, uuid, required): Internal item identifier
- `notifications` (boolean, required): Notification setting (default: false)
- `created` (string, date-time, required): When item was added to watchlist
- `updated` (string, date-time, required): When watchlist item was last updated

**Example Response**:
```json
{
  "shopId": "550e8400-e29b-41d4-a716-446655440000",
  "shopsItemId": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
  "itemId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "notifications": false,
  "created": "2024-01-15T08:00:00Z",
  "updated": "2024-01-15T08:00:00Z"
}
```

**Migration Impact**:
- Frontend applications can now immediately use the returned watchlist item data without making an additional GET request
- The response includes the internal `itemId` which may be useful for tracking
- Applications should update their POST request handlers to process the response body

### Implementation Details

**Backend Changes**:
- Database schema updated to use composite key (`shopId` + `shopsItemId`) as the sort key instead of `created` timestamp
- The `created` timestamp moved to a Local Secondary Index (LSI) named `lsi1` for efficient sorting by creation date
- Service layer methods now accept `shopId` and `shopsItemId` parameters instead of `created` timestamp
- New `WatchlistItemData` type moved to a shared module for reuse across API handlers
- Added `UpdateWatchlistItemCommand` for encapsulating update operations

**Error Code Changes**:
- Error code `WATCHLIST_ENTRY_NOT_FOUND` message updated to reflect new identification method:
  - Old: "There exists no Watchlist-Item that was started being watched on '{timestamp}' for user '{userId}'"
  - New: "There exists no Watchlist-Item for user '{userId}' with Shop-Id '{shopId}' and Shops-Item-Id '{shopsItemId}'"

### Migration Guide

For frontend developers integrating these changes:

1. **DELETE endpoint**:
   - Remove the `created` query parameter from DELETE requests
   - Update URL construction: `DELETE /api/v1/watchlist/${shopId}/${shopsItemId}`
   - Remove error handling for `BAD_QUERY_PARAMETER_VALUE` and `INVALID_RFC3339_TIMESTAMP` related to `created`

2. **PATCH endpoint**:
   - Remove the `created` query parameter from PATCH requests
   - Update URL construction: `PATCH /api/v1/watchlist/${shopId}/${shopsItemId}`
   - Remove error handling for `BAD_QUERY_PARAMETER_VALUE` and `INVALID_RFC3339_TIMESTAMP` related to `created`
   - Request body and response handling remain unchanged

3. **POST endpoint**:
   - Update POST request handlers to process the response body
   - The response now contains the complete watchlist item data including `itemId`, `notifications`, `created`, and `updated`
   - Use the returned data to update your local state without needing an additional GET request

4. **General**:
   - The `created` timestamp is still available in GET responses and can be used for display purposes
   - The uniqueness constraint is now on `(userId, shopId, shopsItemId)` instead of `(userId, created)`
   - A user can only have one watchlist entry per unique item (shopId + shopsItemId combination)

### Removed

No endpoints or data types have been removed. The changes are modifications to existing endpoint signatures and response structures.

## 2025-10-13 - Watchlist Notifications

This update adds notification management capabilities for watchlist items and enriches watchlist item data with additional metadata.

### Added

#### New Endpoint

**PATCH /api/v1/watchlist/{shopId}/{shopsItemId}**
- Updates settings for a specific watchlist item (currently supports toggling notifications)
- Requires the `created` timestamp as a query parameter to identify the exact entry
- Returns 200 OK with updated watchlist item data including core identifiers
- **Authentication**: Required (Cognito JWT)
- **Path Parameters**:
  - `shopId` (required): UUID of the shop
  - `shopsItemId` (required): Shop's item identifier
- **Query Parameters**:
  - `created` (required): RFC3339 timestamp of when the watchlist entry was created
- **Request Body**: `WatchlistItemPatch`
  - `notifications` (optional, boolean): Whether to enable or disable notifications for this item
- **Response Body**: `WatchlistItemPatchResponse`
  - `shopId` (string, uuid, required): Shop identifier
  - `shopsItemId` (string, required): Shop's item identifier
  - `itemId` (string, uuid, required): Internal item identifier
  - `notifications` (boolean, required): Current notification setting
  - `created` (string, date-time, required): When item was added to watchlist
  - `updated` (string, date-time, required): When watchlist item was last updated
- **Headers**:
  - `Authorization` (required): Cognito JWT Bearer token
  - `Last-Modified` (response): RFC3339 timestamp of when the item was last updated

**Example Request**:
```json
{
  "notifications": true
}
```

**Example Response**:
```json
{
  "shopId": "550e8400-e29b-41d4-a716-446655440000",
  "shopsItemId": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
  "itemId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "notifications": true,
  "created": "2024-01-15T08:00:00Z",
  "updated": "2024-01-15T08:30:00Z"
}
```

**Error Responses**:
- `400 BAD_PATH_PARAMETER_VALUE`: Missing or invalid path parameters (shopId, shopsItemId)
- `400 BAD_QUERY_PARAMETER_VALUE`: Missing created timestamp
- `400 INVALID_RFC3339_TIMESTAMP`: Invalid created timestamp format
- `400 INVALID_UUID`: Invalid UUID format for shopId
- `400 BAD_BODY_VALUE`: Missing or invalid request body
- `401 UNAUTHORIZED`: Missing or invalid JWT token
- `404 WATCHLIST_ENTRY_NOT_FOUND`: Watchlist entry not found for the given parameters
- `500 INTERNAL_SERVER_ERROR`: Server error

#### New Data Types

1. **WatchlistItemPatch**
   - Request body schema for PATCH /api/v1/watchlist/{shopId}/{shopsItemId}
   - Properties:
     - `notifications` (boolean, optional): Enable or disable notifications for the watchlist item

2. **WatchlistItemPatchResponse**
   - Response schema for PATCH /api/v1/watchlist/{shopId}/{shopsItemId}
   - Properties:
     - `shopId` (string, uuid, required): Shop identifier
     - `shopsItemId` (string, required): Shop's item identifier
     - `itemId` (string, uuid, required): Internal item identifier
     - `notifications` (boolean, required): Current notification setting
     - `created` (string, date-time, required): When item was added to watchlist
     - `updated` (string, date-time, required): When watchlist item was last updated

### Changed

#### WatchlistItemData (GET /api/v1/watchlist Response)

The `WatchlistItemData` type returned by the GET /api/v1/watchlist endpoint has been enhanced with notification settings and update tracking:

**New Fields Added**:
- `notifications` (boolean, required): Whether notifications are enabled for this watchlist item
  - Default value: `false` for newly created watchlist items
- `updated` (string, date-time, required): RFC3339 timestamp of when the watchlist item was last updated
  - Initially set to the same value as `created` when an item is added to the watchlist
  - Updated whenever watchlist item settings are modified (e.g., via PATCH endpoint)

**Updated Schema**:
```json
{
  "item": {
    // ... GetItemData fields
  },
  "notifications": false,
  "created": "2024-01-15T08:00:00Z",
  "updated": "2024-01-15T08:00:00Z"
}
```

**Impact on Existing Integrations**:
- Frontend applications should update their models to include these new required fields
- The `notifications` field allows users to control whether they receive updates about price changes or availability for watchlist items
- The `updated` field helps track when settings were last modified, useful for sync and conflict resolution

### Migration Guide

For frontend developers integrating these changes:

1. **GET /api/v1/watchlist**:
   - Update your `WatchlistItemData` model to include:
     - `notifications: boolean` (required)
     - `updated: string` (required, RFC3339 format)
   - Handle these fields in your watchlist display components
   - Consider showing notification status in the UI

2. **New PATCH endpoint**:
   - Implement notification toggle functionality using `PATCH /api/v1/watchlist/{shopId}/{shopsItemId}?created={timestamp}`
   - Include the exact `created` timestamp from the watchlist item when making PATCH requests
   - The response provides updated item metadata including the new `updated` timestamp
   - Use the `Last-Modified` response header for optimistic locking if needed

3. **Error Handling**:
   - Add handlers for the new error responses from the PATCH endpoint
   - The `created` timestamp must match exactly - handle `WATCHLIST_ENTRY_NOT_FOUND` errors appropriately

### Removed

No endpoints or fields have been removed in this update.

## 2025-10-08 - Item Enrichment with Shop Information

### Changed

#### GetShopData
- Type `GetShopData` replaced field `url` that contained a single URL with an array `urls` which contains all known URLs for a shop.

#### PUT /api/v1/items - Automatic Shop Enrichment

The PUT items endpoint now automatically enriches items with shop information based on their URLs. This removes the need for clients to provide shop identifiers.

**Request Body Changes (breaking changes)**:
- **Removed**: `shopId` field - No longer required in request body
- **Removed**: `shopName` field - No longer required in request body
- **Shop Enrichment**: Shop information is now automatically determined from the item's `url` field
  - The item's URL must belong to a shop that is already registered in the system
  - If the shop cannot be identified, the item will fail with a `SHOP_NOT_FOUND` error

**Request Body Example**:
```json
{
  "items": [
    {
      "shopsItemId": "item-123",
      "title": {
        "text": "Smartphone Case",
        "language": "en"
      },
      "price": {
        "currency": "EUR",
        "amount": 2999
      },
      "state": "AVAILABLE",
      "url": "https://tech-store.com/items/case-123",
      "images": ["https://tech-store.com/images/case.jpg"]
    }
  ]
}
```

**Response Body Changes (breaking changes)**:
- **Changed**: `unprocessed` field now contains an array of item URLs (strings) instead of ItemKeyData objects
  - Old format: `[{"shopId": "uuid", "shopsItemId": "string"}]`
  - New format: `["https://shop.com/item1", "https://shop.com/item2"]`
  - These are items that could not be processed due to temporary issues and may succeed if retried

- **Added**: `failed` field - A map of item URLs to error codes for items that permanently failed processing
  - Type: Object with URL keys and error code values
  - Example: `{"https://unknown-shop.com/item": "SHOP_NOT_FOUND"}`
  - These items failed permanently and will not succeed without changes

**Response Example**:
```json
{
  "unprocessed": [
    "https://tech-store.com/temporary-issue-item"
  ],
  "failed": {
    "https://unknown-shop.com/item": "SHOP_NOT_FOUND",
    "https://tech-store.com/expensive-item": "MONETARY_AMOUNT_OVERFLOW"
  },
  "skipped": 5
}
```

**New Error Codes**:
- `SHOP_NOT_FOUND`: The shop associated with the item's URL is not registered in the system
- `MONETARY_AMOUNT_OVERFLOW`: The price amount exceeds the maximum supported value during currency conversion
- `ITEM_ENRICHMENT_FAILED`: Failed to enrich the item with additional shop and price information

**Migration Guide**:
1. Remove `shopId` and `shopName` fields from your PUT items requests
2. Ensure the `url` field points to a shop that is registered in the system
3. Handle new error responses in the `failed` field
4. Update error handling for `SHOP_NOT_FOUND`, `MONETARY_AMOUNT_OVERFLOW`, and `ITEM_ENRICHMENT_FAILED` error codes
5. Update processing of `unprocessed` field to handle URL strings instead of ItemKeyData objects

### Removed

No endpoints have been removed in this update.

## 2025-10-02 - Pagination Improvements

### Changed

#### Item Search Endpoints - Search-After Cursor Pagination

The item search endpoints now use cursor-based pagination (search-after pattern) instead of offset/limit pagination:

1. **GET /api/v1/items**
   - Changed from offset/limit pagination to search-after cursor pagination
   - **Query Parameters** (breaking changes):
     - **Removed**: `from` (offset) parameter
     - **Added**: `searchAfter` (optional, string): JSON cursor value from previous response
   - **Response Structure** (breaking changes):
     - Pagination fields are now at the root level (no nested `pagination` object)
     - Fields: `items`, `size`, `total`, `searchAfter`
     - `searchAfter` is returned when more results are available
     - Example response:
       ```json
       {
         "items": [...],
         "size": 21,
         "total": 127,
         "searchAfter": "[2999, \"6ba7b810-9dad-11d1-80b4-00c04fd430c8\"]"
       }
       ```
   - **Previous response structure** (deprecated):
     ```json
     {
       "items": [...],
       "pagination": {
         "from": 0,
         "size": 21,
         "total": 127
       }
     }
     ```

2. **POST /api/v1/items/search**
   - Changed from offset/limit pagination to search-after cursor pagination
   - **Query Parameters** (breaking changes):
     - **Removed**: `from` (offset) parameter
     - **Added**: `searchAfter` (optional, string): JSON cursor value from previous response
   - **Response Structure** (breaking changes):
     - Same as GET /api/v1/items (flattened pagination with `searchAfter`)

#### New Sort Field for Item Searches

- **Added**: `score` sort field for item search endpoints
- Available for both `GET /api/v1/items` and `POST /api/v1/items/search`
- When not specified, `score` is used as the default sort field (sorting by relevance)
- The `sort` query parameter now accepts: `score`, `price`, `updated`, `created`
- **Previous valid values**: `price`, `updated`, `created`

#### New Sort Field for Shop Search

- **Added**: `score` sort field for shop search endpoint
- Available for `POST /api/v1/shops/search`
- When not specified, `score` is used as the default sort field (sorting by relevance)
- The `sort` query parameter now accepts: `score`, `name`, `updated`, `created`
- **Previous valid values**: `name`, `updated`, `created`

#### Pagination Response Structure Changes

The following endpoints now return pagination metadata at the root level instead of in a nested `pagination` object:

1. **GET /api/v1/watchlist**
   - Already used cursor-based pagination, now with flattened structure
   - **Query Parameters** (breaking change):
     - **Renamed**: `from` → `searchAfter` (RFC3339 timestamp)
   - **Response Structure** (breaking change):
     - Pagination fields moved to root level: `items`, `size`, `searchAfter`, `total`
     - `searchAfter` field renamed from `next` and moved to root level
     - Example response:
       ```json
       {
         "items": [...],
         "size": 21,
         "searchAfter": "2024-01-15T08:00:00Z",
         "total": 127
       }
       ```
   - **Previous response structure** (deprecated):
     ```json
     {
       "items": [...],
       "pagination": {
         "from": "2024-01-01T00:00:00Z",
         "size": 21,
         "next": "2024-01-15T08:00:00Z",
         "total": 127
       }
     }
     ```

2. **GET /api/v1/search-filters**
   - Still uses offset/limit pagination
   - **Response Structure** (breaking change):
     - Pagination fields moved to root level: `items`, `from`, `size`, `total`
     - Example response:
       ```json
       {
         "items": [...],
         "from": 0,
         "size": 21,
         "total": 127
       }
       ```
   - **Previous response structure** (deprecated):
     ```json
     {
       "items": [...],
       "pagination": {
         "from": 0,
         "size": 21,
         "total": 127
       }
     }
     ```

3. **POST /api/v1/shops/search**
   - Still uses offset/limit pagination
   - **Response Structure** (breaking change):
     - Same as search filters (flattened pagination structure)

#### New Data Types

1. **ItemSearchResultData**
   - Response type for item search endpoints (GET /api/v1/items, POST /api/v1/items/search)
   - Properties:
     - `items` (array of GetItemData, required): Items in current page
     - `size` (integer, required): Number of items in current page
     - `total` (integer, optional, nullable): Total count of matching items
     - `searchAfter` (string, optional, nullable): Cursor for next page (JSON value)

2. **SearchFilterCollectionData**
   - Response type for GET /api/v1/search-filters
   - Properties (flattened pagination):
     - `items` (array of UserSearchFilterData, required): Search filters
     - `from` (integer, required): Offset
     - `size` (integer, required): Number of items in current page
     - `total` (integer, optional, nullable): Total count

3. **WatchlistCollectionData**
   - Response type for GET /api/v1/watchlist
   - Properties (flattened pagination):
     - `items` (array of WatchlistItemData, required): Watchlist items
     - `size` (integer, required): Number of items in current page
     - `searchAfter` (string, date-time, optional, nullable): Cursor for next page (RFC3339 timestamp)
     - `total` (integer, optional, nullable): Total count

4. **ShopSearchResultData**
   - Response type for POST /api/v1/shops/search
   - Properties (flattened pagination):
     - `items` (array of GetShopData, required): Shops
     - `from` (integer, required): Offset
     - `size` (integer, required): Number of items in current page
     - `total` (integer, optional, nullable): Total count

#### Updated Sort Field Enums

- **SortItemFieldData** enum now includes `score` as a value (and default)
- Values: `score`, `price`, `updated`, `created`

- **SortShopFieldData** enum now includes `score` as a value (and default)
- Values: `score`, `name`, `updated`, `created`

#### New Error Code

- `INVALID_JSON`: Returned when the `searchAfter` parameter contains invalid JSON

### Migration Guide

For frontend developers integrating these changes:

1. **Item Search Endpoints** (GET /api/v1/items, POST /api/v1/items/search):
   - Replace `from` query parameter with `searchAfter`
   - Read pagination fields from root level instead of `response.pagination.*`
   - Use `response.searchAfter` (if present) for the next page request
   - The `total` field may be `null` in some cases

2. **Watchlist Endpoint** (GET /api/v1/watchlist):
   - **Rename query parameter**: `from` → `searchAfter`
   - Read pagination fields from root level instead of `response.pagination.*`
   - **Field renamed**: `pagination.next` → `searchAfter` (at root level)
   - Use `response.searchAfter` (if present) for the next page request

3. **Search Filters Endpoint** (GET /api/v1/search-filters):
   - Read pagination fields from root level instead of `response.pagination.*`
   - Continue using `from` and `size` query parameters (no change)

4. **Shop Search Endpoint** (POST /api/v1/shops/search):
   - Read pagination fields from root level instead of `response.pagination.*`
   - Continue using `from` and `size` query parameters (no change)
   - The `score` sort field is now available and is the default

5. **Sort Fields**:
   - The `score` sort field is now available for both item search and shop search endpoints
   - When not specifying a sort field, results are sorted by relevance score (default)

### Removed

No endpoints or features have been removed in this update. The changes are modifications to existing endpoints.

## 2025-09-30 - Watchlist Feature

### Added

Three new endpoints have been added to support the user item watchlist feature:

#### New Endpoints

1. **GET /api/v1/watchlist**
   - Retrieves all items in the authenticated user's watchlist
   - Uses cursor-based pagination with RFC3339 timestamps (search-after pattern)
   - Supports sorting by creation date (when item was added to watchlist)
   - Supports language selection via `Accept-Language` header
   - Supports currency selection via `currency` query parameter
   - Returns items with full item details and the timestamp when added to watchlist
   - **Authentication**: Required (Cognito JWT)
   - **Query Parameters**:
     - `currency` (optional): Currency for price display (default: EUR)
     - `sort` (optional): Field to sort by (only `created` supported)
     - `order` (optional): Sort order `asc` or `desc`
     - `from` (optional): RFC3339 timestamp cursor for pagination
     - `size` (optional): Number of items per page (min: 1, max: 100, default: 21)
   - **Headers**:
     - `Accept-Language` (optional): Language for localized content (de, en, fr, es)
     - `Authorization` (required): Cognito JWT Bearer token

2. **POST /api/v1/watchlist**
   - Adds an item to the authenticated user's watchlist
   - Request body must contain `shopId` and `shopsItemId`
   - Returns 201 Created with `Location` header pointing to the resource
   - **Authentication**: Required (Cognito JWT)
   - **Request Body**: `ItemKeyData` (shopId, shopsItemId)
   - **Headers**:
     - `Authorization` (required): Cognito JWT Bearer token

3. **DELETE /api/v1/watchlist/{shopId}/{shopsItemId}**
   - Removes a specific item from the authenticated user's watchlist
   - Requires the `created` timestamp as a query parameter to identify the exact entry
   - Returns 204 No Content on success
   - **Authentication**: Required (Cognito JWT)
   - **Path Parameters**:
     - `shopId` (required): UUID of the shop
     - `shopsItemId` (required): Shop's item identifier
   - **Query Parameters**:
     - `created` (required): RFC3339 timestamp of when the watchlist entry was created
   - **Headers**:
     - `Authorization` (required): Cognito JWT Bearer token

#### New Data Types

1. **ItemKeyData**
   - Object identifying an item by shop ID and shop's item ID
   - Properties:
     - `shopId` (string, uuid, required): Shop identifier
     - `shopsItemId` (string, required): Shop's item identifier

2. **WatchlistItemData**
   - Object representing a watchlist entry
   - Properties:
     - `item` (GetItemData, required): Full item details
     - `created` (string, date-time, required): RFC3339 timestamp when added to watchlist

3. **WatchlistCollectionData**
   - Paginated collection of watchlist items using cursor-based pagination
   - Properties:
     - `items` (array of WatchlistItemData, required): Watchlist items in current page
     - `pagination` (PaginationDataDateTime, required): Pagination metadata

4. **PaginationDataDateTime**
   - Pagination metadata for cursor-based pagination using timestamps
   - Properties:
     - `from` (string, date-time, required): Starting cursor timestamp
     - `size` (integer, required): Number of items in current page
     - `total` (integer, optional, nullable): Total count (may not be available)
     - `next` (string, date-time, optional, nullable): Next page cursor

5. **SortWatchlistItemFieldData**
   - Enum for watchlist sorting fields
   - Values: `created`

#### New Error Codes

- `INVALID_RFC3339_TIMESTAMP`: Returned when an RFC3339 timestamp parameter is malformed
- `WATCHLIST_ENTRY_NOT_FOUND`: Returned when attempting to delete a non-existent watchlist entry

### Changed

#### Generalized type `UnprocessedPutItem`
- The type `UnprocessedPutItem` has been generalized to `ItemKeyData`. It's a composite key for identifying an item.
- `UnprocessedPutItem` therefore no longer exists. `ItemKeyData` is used now. Their payload does **not** differ.

#### Pagination Pattern
- Watchlist endpoints use cursor-based pagination with RFC3339 timestamps instead of offset-based pagination
- The `from` query parameter accepts an RFC3339 timestamp
- The response includes a `next` field in pagination metadata when more results are available
- The `total` field in pagination metadata is optional for cursor-based pagination
- The other endpoints using pagination are unaffected and still rely on Offset/Limit-Pagination.

### Removed

No endpoints or features have been removed in this update.
