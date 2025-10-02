# Changelog

All notable changes to the Blitzfilter Backend API will be documented in this file.

This changelog is for internal communication between frontend and backend teams. It documents what is new, what has changed, and what has been removed for each API update.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

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

#### Pagination Response Structure Changes

The following endpoints now return pagination metadata at the root level instead of in a nested `pagination` object:

1. **GET /api/v1/search-filters**
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

2. **POST /api/v1/shops/search**
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

3. **ShopSearchResultData**
   - Response type for POST /api/v1/shops/search
   - Properties (flattened pagination):
     - `items` (array of GetShopData, required): Shops
     - `from` (integer, required): Offset
     - `size` (integer, required): Number of items in current page
     - `total` (integer, optional, nullable): Total count

#### Updated Sort Field Enum

- **SortItemFieldData** enum now includes `score` as a value
- Values: `score`, `price`, `updated`, `created`
- `score` is the default when not specified

#### New Error Code

- `INVALID_JSON`: Returned when the `searchAfter` parameter contains invalid JSON

### Migration Guide

For frontend developers integrating these changes:

1. **Item Search Endpoints** (GET /api/v1/items, POST /api/v1/items/search):
   - Replace `from` query parameter with `searchAfter`
   - Read pagination fields from root level instead of `response.pagination.*`
   - Use `response.searchAfter` (if present) for the next page request
   - The `total` field may be `null` in some cases

2. **Search Filters Endpoint** (GET /api/v1/search-filters):
   - Read pagination fields from root level instead of `response.pagination.*`
   - Continue using `from` and `size` query parameters (no change)

3. **Shop Search Endpoint** (POST /api/v1/shops/search):
   - Read pagination fields from root level instead of `response.pagination.*`
   - Continue using `from` and `size` query parameters (no change)

4. **Sort Field**:
   - The `score` sort field is now available and is the default
   - When not specifying a sort field, results are sorted by relevance score

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
