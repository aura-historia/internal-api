# Changelog

All notable changes to the Blitzfilter Backend API will be documented in this file.

This changelog is for internal communication between frontend and backend teams. It documents what is new, what has changed, and what has been removed for each API update.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## 2025-10-08 - Item Enrichment with Shop Information

### Changed

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
