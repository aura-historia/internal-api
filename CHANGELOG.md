# Changelog

All notable changes to the Aura-Historia Backend API will be documented in this file.

This changelog is for internal communication between frontend and backend teams. It documents what is new, what has changed, and what has been removed for each API update.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## 2026-02-10 - Product Category Classification

This update adds automatic level-one category classification to product data. Products are now classified into top-level categories (e.g., "musical-instruments", "antique-furniture", "antique-clocks") through an event-driven classification pipeline. Category information is returned in all product responses with localized display names supporting German, English, French, and Spanish.

### Added

**Product Category Fields** - Two new optional fields added to product responses:

**GetProductData Schema**:
- **`categoryId`** (string, nullable, pattern: `^[a-z0-9]+(-[a-z0-9]+)*$`):
  - Kebab-case identifier for the level-one category the product has been classified into
  - Categories are automatically assigned by the backend classification system
  - Examples: `"musical-instruments"`, `"antique-furniture"`, `"antique-clocks"`, `"archaeological-artifacts"`
  - Optional field - will be `null` if product has not yet been classified
  
- **`category`** (LocalizedTextData, nullable):
  - Localized display name for the product's level-one category
  - Language matches the product's primary content language
  - Supports: German (de), English (en), French (fr), Spanish (es)
  - Examples:
    - German: `{"text": "Historische Musikinstrumente", "language": "de"}`
    - English: `{"text": "Antique Musical Instruments", "language": "en"}`
    - French: `{"text": "Instruments de musique anciens", "language": "fr"}`
    - Spanish: `{"text": "Instrumentos musicales antiguos", "language": "es"}`
  - Optional field - will be `null` if product has not yet been classified

**GetProductSummaryData Schema** - Same category fields as above:
- **`categoryId`** (string, nullable)
- **`category`** (LocalizedTextData, nullable)

**Example Response with Category Data**:
```json
{
  "productId": "550e8400-e29b-41d4-a716-446655440000",
  "shopName": "My Shop",
  "shopType": "COMMERCIAL_DEALER",
  "categoryId": "musical-instruments",
  "category": {
    "text": "Antique Musical Instruments",
    "language": "en"
  },
  "title": {
    "text": "Vintage Violin",
    "language": "en"
  }
}
```

### Changed

**All Product Endpoints** now include category fields in their responses:
- `GET /api/v1/shops/{shopId}/products/{shopsProductId}` - Returns `GetProductData` with category fields
- `GET /api/v1/by-slug/shops/{shopSlugId}/products/{productSlugId}` - Returns `GetProductData` with category fields
- `GET /api/v1/shops/{shopId}/products/{shopsProductId}/similar` - Returns array of `GetProductSummaryData` with category fields
- `GET /api/v1/products` - Returns array of `GetProductSummaryData` with category fields
- `GET /api/v1/products/search` - Returns array of `GetProductSummaryData` with category fields

**Backward Compatibility**: These changes are fully backward compatible as both new fields are optional (nullable). Existing integrations will continue to work without modification.

**Implementation Notes**:
- Classification is event-driven and asynchronous - newly added products may not have category data immediately
- Categories are assigned at the level-one (top-level) only - no sub-categories at this time
- The classification system uses machine learning and rule-based approaches for automatic categorization
- Category display names are stored and returned in the language that best matches the product's content

## 2026-02-07 - Restructure Product Data Response Format

This update restructures the product data response format to better organize pricing, origin year, and auction information into composite objects. This provides a more cohesive API structure and reduces field proliferation at the top level.

### Added

**New Schema - AuctionData**:
- **Properties**:
  - `start` (string, date-time, nullable): Start datetime of the auction window in RFC3339 format. Optional field indicating when bidding begins or when the item will be auctioned.
  - `end` (string, date-time, nullable): End datetime of the auction window in RFC3339 format. Optional field indicating when bidding ends or when the auction session concludes.
- At least one of the fields (start or end) is present when the AuctionData object is included in a response.
- Both fields use RFC3339 datetime format (e.g., "2025-05-01T12:00:00Z")

**New Schema - PricingData**:
- **Properties**:
  - `offer` (PriceData, nullable): The actual offer or asking price for the product. This is the price at which the product is currently being sold.
  - `estimate` (PriceEstimateData, nullable): Optional price estimate range, common for auction items.
- Combines offer price and estimates into a single cohesive pricing structure.

**New Schema - PriceEstimateData**:
- **Properties**:
  - `min` (PriceData, nullable): Optional minimum estimated price for the product.
  - `max` (PriceData, nullable): Optional maximum estimated price for the product.
- Encapsulates price estimate range information.

**New Schema - OriginYearData**:
- **Properties**:
  - `min` (integer, nullable): Lower end of the year range when the antique is estimated to have originated.
  - `year` (integer, nullable): Exact year the antique is estimated to have originated.
  - `max` (integer, nullable): Upper end of the year range when the antique is estimated to have originated.
- At least one of the fields will be present when this object is included.
- Can represent either an exact year or a year range.

### Changed

**GetProductData Schema** - Major restructuring of product fields:

1. **Price Structure Changed**:
   - **Removed**: `price`, `priceEstimateMin`, `priceEstimateMax` (three separate fields)
   - **Added**: `price` (PricingData, nullable) - Single field containing nested offer and estimate data
   - **Example Before**:
     ```json
     {
       "price": { "currency": "EUR", "amount": 2999 },
       "priceEstimateMin": { "currency": "EUR", "amount": 2500 },
       "priceEstimateMax": { "currency": "EUR", "amount": 3500 }
     }
     ```
   - **Example After**:
     ```json
     {
       "price": {
         "offer": { "currency": "EUR", "amount": 2999 },
         "estimate": {
           "min": { "currency": "EUR", "amount": 2500 },
           "max": { "currency": "EUR", "amount": 3500 }
         }
       }
     }
     ```

2. **Origin Year Structure Changed**:
   - **Removed**: `originYearMin`, `originYear`, `originYearMax` (three separate fields)
   - **Added**: `originYear` (OriginYearData, nullable) - Single field containing nested min/year/max data
   - **Example Before**:
     ```json
     {
       "originYear": 1837
     }
     ```
   - **Example After**:
     ```json
     {
       "originYear": { "year": 1837 }
     }
     ```

3. **Auction Fields Restructured**:
   - **Removed**: `auctionStart`, `auctionEnd` (two separate fields)
   - **Added**: `auction` (AuctionData, nullable) - Single field containing nested start and end data
   - **Example Before**:
     ```json
     {
       "auctionStart": "2025-05-01T12:00:00Z",
       "auctionEnd": "2025-05-10T12:00:00Z"
     }
     ```
   - **Example After**:
     ```json
     {
       "auction": {
         "start": "2025-05-01T12:00:00Z",
         "end": "2025-05-10T12:00:00Z"
       }
     }
     ```

4. **Metadata Fields Now Required**:
   - `authenticity`, `condition`, `provenance`, `restoration` are now required fields (always present with default value "UNKNOWN" when not specified)
   - Previously these fields were nullable and could be absent from responses

**GetProductSummaryData Schema** - Added auction field:
- **New Property**:
  - `auction` (AuctionData, nullable): Optional auction time window information. Only present for products from auction houses with scheduled auction times. Contains start and/or end timestamps for the auction.
- This field is skipped in JSON responses when null
- The GetProductSummaryData description has been updated to reflect that auction times are now included (previously listed as excluded metadata)

**Affected Endpoints**:
All endpoints returning `GetProductData` or `PersonalizedGetProductData` now use the restructured format:
- **GET /api/v1/shops/{shopId}/products/{shopsProductId}** - Returns PersonalizedGetProductData
- **GET /api/v1/by-slug/shops/{shopSlugId}/products/{productSlugId}** - Returns PersonalizedGetProductData

All endpoints returning `GetProductSummaryData` or `PersonalizedGetProductSummaryData` now include the auction field:
- **GET /api/v1/shops/{shopId}/products/{shopsProductId}/similar** - Returns array of PersonalizedGetProductSummaryData
- **POST /api/v1/products/search** - Returns PersonalizedProductSearchResultData containing PersonalizedGetProductSummaryData items

## 2026-01-25 - Restructure REST API Resource Paths

This update restructures API resource paths to follow a more hierarchical and RESTful pattern. Product endpoints now consistently use the `/shops/{shopId}/products/...` path structure, and shop identifier endpoints have been split into separate, more explicit endpoints based on the type of identifier (ID, domain, or slug).

### Changed

**Product Endpoints** - Restructured to nest under shops resource:

- **GET /api/v1/products/{shopId}/{shopsProductId}** → **GET /api/v1/shops/{shopId}/products/{shopsProductId}**
  - Path structure now clearly shows the hierarchical relationship between shops and products
  - All parameters, headers, responses, and behavior remain identical
  
- **GET /api/v1/products/by-slug/{shopSlugId}/{productSlugId}** → **GET /api/v1/by-slug/shops/{shopSlugId}/products/{productSlugId}**
  - Path structure now clearly shows the hierarchical relationship between shops and products
  - The `/by-slug/` prefix is moved to the beginning to group all slug-based lookups
  - All parameters, headers, responses, and behavior remain identical

- **GET /api/v1/products/{shopId}/{shopsProductId}/similar** → **GET /api/v1/shops/{shopId}/products/{shopsProductId}/similar**
  - Path structure now clearly shows the hierarchical relationship between shops and products
  - All parameters, headers, responses, and behavior remain identical

- **GET /api/v1/products/{shopId}/{shopsProductId}/history** → **GET /api/v1/shops/{shopId}/products/{shopsProductId}/history**
  - Path structure now clearly shows the hierarchical relationship between shops and products
  - All parameters, headers, responses, and behavior remain identical

**Shop Endpoints** - Split polymorphic identifier endpoint into explicit endpoints:

- **GET /api/v1/shops/{shopIdentifier}** has been split into two separate endpoints:
  - **GET /api/v1/shops/{shopId}** - Retrieve shop by UUID
    - **Path Parameters**:
      - `shopId` (string, uuid, required): Unique identifier of the shop
    - Previously this was accessed via `/api/v1/shops/{shopIdentifier}` with a UUID value
    - Response structure, headers, and behavior remain identical
    - **Error Response Changes**:
      - Field name in errors changed from `shopIdentifier` to `shopId`
      - Error code for invalid format changed from `INVALID_SHOP_IDENTIFIER` to `INVALID_UUID`
      
  - **GET /api/v1/by-domain/shops/{shopDomain}** - Retrieve shop by domain (NEW)
    - **Path Parameters**:
      - `shopDomain` (string, required): Domain associated with the shop (e.g., "tech-store.com")
      - Domain is normalized (lowercased, www prefix removed)
      - Pattern: `^[a-z0-9]+([-.][a-z0-9]+)*\.[a-z]{2,}$`
    - Previously this was accessed via `/api/v1/shops/{shopIdentifier}` with a domain value
    - Response structure, headers, and behavior remain identical
    - **Error Response Changes**:
      - Field name in errors changed from `shopIdentifier` to `shopDomain`
      - Error code for invalid format changed from `INVALID_SHOP_IDENTIFIER` to `INVALID_DOMAIN`

- **GET /api/v1/shops/by-slug/{shopSlugId}** → **GET /api/v1/by-slug/shops/{shopSlugId}**
  - The `/by-slug/` prefix is moved to the beginning to group all slug-based lookups
  - All parameters, headers, responses, and behavior remain identical

- **PATCH /api/v1/shops/{shopIdentifier}** has been split into two separate endpoints:
  - **PATCH /api/v1/shops/{shopId}** - Update shop by UUID
    - **Path Parameters**:
      - `shopId` (string, uuid, required): Unique identifier of the shop
    - Previously this was accessed via `/api/v1/shops/{shopIdentifier}` with a UUID value
    - Request body, response structure, headers, and behavior remain identical
    - **Error Response Changes**:
      - Field name in errors changed from `shopIdentifier` to `shopId`
      - Error code for invalid format changed from `INVALID_SHOP_IDENTIFIER` to `INVALID_UUID`
      
  - **PATCH /api/v1/by-domain/shops/{shopDomain}** - Update shop by domain (NEW)
    - **Path Parameters**:
      - `shopDomain` (string, required): Domain associated with the shop (e.g., "tech-store.com")
      - Domain is normalized (lowercased, www prefix removed)
      - Pattern: `^[a-z0-9]+([-.][a-z0-9]+)*\.[a-z]{2,}$`
    - Previously this was accessed via `/api/v1/shops/{shopIdentifier}` with a domain value
    - Request body, response structure, headers, and behavior remain identical
    - **Error Response Changes**:
      - Field name in errors changed from `shopIdentifier` to `shopDomain`
      - Error code for invalid format changed from `INVALID_SHOP_IDENTIFIER` to `INVALID_DOMAIN`

### Removed

- **GET /api/v1/shops/{shopIdentifier}** - Removed polymorphic endpoint
  - Use `GET /api/v1/shops/{shopId}` for UUID-based lookup
  - Use `GET /api/v1/by-domain/shops/{shopDomain}` for domain-based lookup
  
- **PATCH /api/v1/shops/{shopIdentifier}** - Removed polymorphic endpoint
  - Use `PATCH /api/v1/shops/{shopId}` for UUID-based updates
  - Use `PATCH /api/v1/by-domain/shops/{shopDomain}` for domain-based updates

## 2026-01-24 - Split Product History into Dedicated Endpoint

This update separates product history retrieval from the main product endpoint into a dedicated history endpoint. This change provides better separation of concerns and allows the history endpoint to return a cleaner array structure instead of nesting history data within the product response.

### Added

**GET /api/v1/products/{shopId}/{shopsProductId}/history** - New endpoint to retrieve product event history:
- **Path Parameters**:
  - `shopId` (string, uuid, required): Unique identifier of the shop
  - `shopsProductId` (string, required): Shop's unique identifier for the product
- **Query Parameters**:
  - `currency` (CurrencyData, optional): Currency for price display in event payloads (default: EUR)
- **Headers**:
  - `Accept-Language` (string, optional): Preferred language for localized content
- **Response** (200 OK):
  - Returns an array of `GetProductEventData` objects representing the product's event history
  - Events are ordered chronologically
  - Each event contains:
    - `eventType` (ProductEventTypeData): Type of event (CREATED, STATE_LISTED, STATE_AVAILABLE, STATE_RESERVED, STATE_SOLD, STATE_REMOVED, STATE_UNKNOWN, PRICE_DISCOVERED, PRICE_DROPPED, PRICE_INCREASED, PRICE_REMOVED)
    - `productId` (string, uuid): Unique internal product identifier
    - `eventId` (string, uuid): Unique event identifier
    - `shopId` (string, uuid): Shop identifier
    - `shopsProductId` (string): Shop's product identifier
    - `payload` (ProductEventPayloadData): Event-specific payload with details
    - `timestamp` (string, date-time): When the event occurred (RFC3339 format)
- **Response Headers**:
  - `Content-Language`: Language of returned content
  - `Access-Control-Allow-Origin`: CORS header
- **Error Responses**:
  - 400 Bad Request: Invalid parameters (missing shopId or shopsProductId)
  - 404 Not Found: Product not found
  - 500 Internal Server Error: Server error

**Example Response**:
```json
[
  {
    "eventType": "CREATED",
    "productId": "550e8400-e29b-41d4-a716-446655440000",
    "eventId": "650e8400-e29b-41d4-a716-446655440001",
    "shopId": "550e8400-e29b-41d4-a716-446655440000",
    "shopsProductId": "6ba7b810",
    "payload": {
      "price": {
        "currency": "EUR",
        "amount": 3000
      },
      "state": "LISTED"
    },
    "timestamp": "2024-01-01T10:00:00Z"
  },
  {
    "eventType": "STATE_AVAILABLE",
    "productId": "550e8400-e29b-41d4-a716-446655440000",
    "eventId": "650e8400-e29b-41d4-a716-446655440002",
    "shopId": "550e8400-e29b-41d4-a716-446655440000",
    "shopsProductId": "6ba7b810",
    "payload": {
      "oldState": "LISTED",
      "newState": "AVAILABLE"
    },
    "timestamp": "2024-01-02T14:30:00Z"
  },
  {
    "eventType": "PRICE_DROPPED",
    "productId": "550e8400-e29b-41d4-a716-446655440000",
    "eventId": "650e8400-e29b-41d4-a716-446655440003",
    "shopId": "550e8400-e29b-41d4-a716-446655440000",
    "shopsProductId": "6ba7b810",
    "payload": {
      "oldPrice": {
        "currency": "EUR",
        "amount": 3000
      },
      "newPrice": {
        "currency": "EUR",
        "amount": 2999
      }
    },
    "timestamp": "2024-01-15T09:15:00Z"
  }
]
```

### Changed

**GET /api/v1/products/{shopId}/{shopsProductId}** - Removed history query parameter:
- The `history` query parameter has been removed
- Previously: `?history=true` would include history events in the response
- Now: Use the dedicated `/history` endpoint to retrieve product event history
- The response no longer includes a `history` field in the product data

**GET /api/v1/products/by-slug/{shopSlugId}/{productSlugId}** - Removed history query parameter:
- The `history` query parameter has been removed
- Previously: `?history=true` would include history events in the response
- Now: Use the dedicated `/history` endpoint with product IDs to retrieve event history
- The response no longer includes a `history` field in the product data

### Removed

**GetProductData.history** - Field removed from product response:
- Previously: `history` (array of GetProductEventData, optional, nullable)
- Now: Field no longer exists in the schema
- To retrieve product history, use the new dedicated endpoint: `GET /api/v1/products/{shopId}/{shopsProductId}/history`

**Query Parameter `history`** - Removed from product retrieval endpoints:
- Previously accepted on: `GET /api/v1/products/{shopId}/{shopsProductId}` and `GET /api/v1/products/by-slug/{shopSlugId}/{productSlugId}`
- The parameter and its validation error responses have been removed

### Rationale

This change provides:
- **Better Separation of Concerns**: Product details and event history are distinct resources with dedicated endpoints
- **Cleaner Response Structure**: History endpoint returns a direct array instead of nested data
- **Performance Optimization**: Product retrieval no longer carries optional heavy history data
- **API Design Consistency**: Follows RESTful principles with resource-specific endpoints

**Migration Guide**:
- **Before**: `GET /api/v1/products/{shopId}/{shopsProductId}?history=true`
- **After**: First call `GET /api/v1/products/{shopId}/{shopsProductId}` to get product details, then call `GET /api/v1/products/{shopId}/{shopsProductId}/history` to get event history if needed

## 2026-01-24 - Shop Name Update Restriction and Slug Uniqueness Enforcement

This update removes the ability to update a shop's name after creation and adds enforcement of shop slug uniqueness during shop creation. These changes ensure shop slug identifiers remain stable and unique throughout the shop's lifetime.

### Changed

**PATCH /api/v1/shops/{shopIdentifier}** - Shop name can no longer be updated:
- The `name` field has been removed from the request body schema (`PatchShopData`)
- Shop names are now immutable after creation since they determine the shop's slug identifier
- Only `shopType`, `domains`, and `image` fields can be updated
- Attempting to update the name will have no effect (field is ignored if provided)

**POST /api/v1/shops** - Enhanced uniqueness validation:
- Now validates shop slug uniqueness in addition to domain uniqueness
- Returns HTTP 409 Conflict when attempting to create a shop with a name that generates a slug already in use
- Two distinct conflict scenarios:
  - **Slug conflict**: Returns error "Shop with name '{name}' exists already - the shop-slug '{slug}' is already registered."
  - **Domain conflict**: Returns error "Shop with name '{name}' exists already - a domain of the shop is already registered."

### Removed

**PatchShopData.name** - Field removed from shop update request:
- Previously: `name` (string, optional, nullable)
- Now: Field no longer exists in the schema
- Rationale: Shop names determine slug identifiers which are used in URLs and must remain stable

### Rationale

This change ensures:
- **URL Stability**: Shop slugs in URLs (e.g., `/api/v1/shops/by-slug/tech-store-premium`) remain valid and consistent
- **Slug Uniqueness**: Each shop has a unique slug identifier, preventing confusion and routing issues
- **Data Integrity**: Shop name and slug remain aligned throughout the shop's lifetime
- **API Consistency**: The slug-based endpoints work reliably without handling name/slug mismatches

**Migration Note**: Existing shops retain their names and slugs. Only future updates are affected by this restriction. If a shop name change is absolutely necessary, it must be handled as a special administrative operation outside the normal API flow.

## 2026-01-24 - Separate Product DTOs for Search and Similar Products

This update introduces a new lightweight product summary data type (`GetProductSummaryData`) for use in search results and similar products listings. This reduces response payload size by excluding detailed metadata fields that are not needed in list views.

### Added

**GetProductSummaryData** - New lightweight product data type with the following fields:
- `productId` (string, uuid, required): Unique internal identifier for the product
- `productSlugId` (string, required): Human-readable slug identifier (e.g., "amazing-product-fa87c4")
- `shopSlugId` (string, required): Human-readable shop slug (e.g., "tech-store-premium")
- `eventId` (string, uuid, required): Unique identifier for current product state/version
- `shopId` (string, uuid, required): Unique shop identifier
- `shopsProductId` (string, required): Shop's unique product identifier
- `shopName` (string, required): Display name of the shop
- `shopType` (ShopTypeData, required): Type of shop (e.g., DEALER, AUCTION_HOUSE)
- `title` (LocalizedTextData, required): Localized product title
- `price` (PriceData, optional): Product price
- `state` (ProductStateData, required): Current product state
- `url` (string, uri, required): URL to product on shop's website
- `images` (array of ProductImageData, required): Product images with prohibited content classification
- `created` (string, date-time, required): When product was first created (RFC3339)
- `updated` (string, date-time, required): When product was last updated (RFC3339)

**PersonalizedGetProductSummaryData** - Wrapper combining `GetProductSummaryData` with optional user state:
- `item` (GetProductSummaryData, required): The product summary data
- `userState` (ProductUserStateData, optional): User-specific state when authenticated

### Changed

**GET /api/v1/products/search** - Response now uses `PersonalizedGetProductSummaryData`:
- Response type changed from `PersonalizedProductSearchResultData<PersonalizedGetProductData>` to `PersonalizedProductSearchResultData<PersonalizedGetProductSummaryData>`
- Each product in search results now excludes: `description`, `priceEstimateMin`, `priceEstimateMax`, `originYear`, `originYearMin`, `originYearMax`, `authenticity`, `condition`, `provenance`, `restoration`, `auctionStart`, `auctionEnd`, `history`
- All core product information remains available (id, title, price, state, images, timestamps)

**GET /api/v1/products/{shopId}/{shopsProductId}/similar** - Response now uses `PersonalizedGetProductSummaryData`:
- Response type changed from `Array<PersonalizedGetProductData>` to `Array<PersonalizedGetProductSummaryData>`
- Each similar product now excludes the same detailed fields as search results (listed above)
- Reduces payload size for similar product suggestions while maintaining essential product information

### Rationale

The new `GetProductSummaryData` type is specifically designed for list/summary views where detailed metadata is not needed. This:
- **Improves Performance**: Significantly reduces response payload size for endpoints that return multiple products
- **Better Separation**: Distinguishes between detailed product views (single product endpoint) and summary views (search/similar)
- **Maintains Compatibility**: The full `GetProductData` remains unchanged and continues to be used for single product retrieval

To get full product details including description, estimates, origin years, authenticity, condition, provenance, restoration, auction times, and history, use the single product endpoints:
- `GET /api/v1/products/{shopId}/{shopsProductId}`
- `GET /api/v1/products/by-slug/{shopSlugId}/{productSlugId}`

## 2026-01-24 - Add shop-type for auction-platforms

This update adds a new shop-type for auction-platforms. 

### Added

- `ShopType::AUCTION_PLATFORM`

## 2026-01-21 - Human-Readable Slug Identifiers

This update introduces human-readable slug identifiers for products and shops, providing SEO-friendly and user-friendly URLs. The new slug-based endpoints allow accessing resources using kebab-case identifiers instead of UUIDs, while maintaining backward compatibility with existing ID-based endpoints.

### Added

#### New Endpoints

**GET /api/v1/products/by-slug/{shopSlugId}/{productSlugId}**:
- Retrieves a single product using human-readable slug identifiers
- **Path Parameters**:
  - `shopSlugId` (string, required): Kebab-case shop identifier (e.g., "tech-store-premium", "christies")
    - Pattern: `^[a-z0-9]+(-[a-z0-9]+)*$`
    - Derived from shop name, normalized to lowercase with hyphens
  - `productSlugId` (string, required): Kebab-case product identifier with 6-character hex suffix
    - Pattern: `^[a-z0-9]+(-[a-z0-9]+)*-[a-f0-9]{6}$`
    - Format: `{product-title}-{6-char-hex}`
    - Example: "amazing-product-fa87c4"
- **Query Parameters**: Same as existing endpoint (`currency`, `history`)
- **Headers**: Same as existing endpoint (`Accept-Language`, optional `Authorization`)
- **Response**: Returns `PersonalizedGetProductData` (200) with product details including new slug fields
- **Authentication**: Optional (supports both authenticated and anonymous access)
- **Behavior**: Identical to GET /api/v1/products/{shopId}/{shopsProductId} but uses slug identifiers

**GET /api/v1/shops/by-slug/{shopSlugId}**:
- Retrieves shop details using human-readable slug identifier
- **Path Parameters**:
  - `shopSlugId` (string, required): Kebab-case shop identifier
    - Pattern: `^[a-z0-9]+(-[a-z0-9]+)*$`
    - Examples: "tech-store-premium", "christies", "sothebys"
- **Response**: Returns `GetShopData` (200) with complete shop information including new slug field
- **Behavior**: Identical to GET /api/v1/shops/{shopIdentifier} but specifically uses slug identifier

#### Schema Changes

**GetProductData**:
Added two new required fields to the product data response:

```json
{
  "productId": "550e8400-e29b-41d4-a716-446655440000",
  "productSlugId": "amazing-product-fa87c4",
  "shopSlugId": "tech-store-premium",
  "eventId": "550e8400-e29b-41d4-a716-446655440001",
  "shopId": "550e8400-e29b-41d4-a716-446655440000",
  "shopsProductId": "6ba7b810",
  ...
}
```

**New Fields**:
- `productSlugId` (string, required):
  - Human-readable product identifier with unique suffix
  - Pattern: `^[a-z0-9]+(-[a-z0-9]+)*-[a-f0-9]{6}$`
  - Format: Kebab-case product title + 6-character hexadecimal suffix
  - Example: "vintage-watch-3a2f9b"
  - The suffix ensures uniqueness when multiple products have similar titles
  
- `shopSlugId` (string, required):
  - Human-readable shop identifier
  - Pattern: `^[a-z0-9]+(-[a-z0-9]+)*$`
  - Format: Kebab-case shop name (no suffix)
  - Example: "tech-store-premium"
  - Derived from shop name by converting to lowercase and replacing spaces/special characters with hyphens

**GetShopData**:
Added one new required field to the shop data response:

```json
{
  "shopId": "550e8400-e29b-41d4-a716-446655440000",
  "shopSlugId": "tech-store-premium",
  "name": "Tech Store Premium",
  "shopType": "COMMERCIAL_DEALER",
  ...
}
```

**New Field**:
- `shopSlugId` (string, required):
  - Human-readable shop identifier
  - Pattern: `^[a-z0-9]+(-[a-z0-9]+)*$`
  - Format: Kebab-case shop name
  - Example: "tech-store-premium" or "christies"

#### Affected Endpoints (Schema Updates)

All endpoints that return product or shop data now include the new slug fields:

**Product Endpoints**:
- GET /api/v1/products/{shopId}/{shopsProductId}
- GET /api/v1/products/by-slug/{shopSlugId}/{productSlugId} (new)
- GET /api/v1/products/{shopId}/{shopsProductId}/similar
- POST /api/v1/products/search
- GET /api/v1/me/watchlist

**Shop Endpoints**:
- GET /api/v1/shops/{shopIdentifier}
- GET /api/v1/shops/by-slug/{shopSlugId} (new)
- POST /api/v1/shops
- PATCH /api/v1/shops/{shopIdentifier}
- POST /api/v1/shops/search

### Technical Details

**Slug Generation**:
- Shop slugs: Generated from shop name using slug library (kebab-case, no suffix)
  - Example: "Tech Store Premium" → "tech-store-premium"
  - Example: "Christie's" → "christies"
- Product slugs: Generated from product title + UUID-based 6-character hex suffix
  - Example: "Amazing Product" + UUID → "amazing-product-fa87c4"
  - The suffix ensures uniqueness across products with similar titles

**Backward Compatibility**:
- All existing ID-based endpoints remain unchanged and fully functional
- New slug-based endpoints provide alternative access patterns
- Clients can choose to use either UUID-based or slug-based endpoints
- All responses now include both ID and slug fields for maximum flexibility

**Error Codes**:
- Same error codes apply to slug-based endpoints as ID-based endpoints
- 400: Missing or invalid slug parameters (BAD_PATH_PARAMETER_VALUE)
- 404: Product or shop not found (PRODUCT_NOT_FOUND, SHOP_NOT_FOUND)
- 500: Internal server error (INTERNAL_SERVER_ERROR)

**Use Cases**:
- SEO-friendly URLs for web applications
- User-shareable links with readable identifiers
- Improved user experience with meaningful URLs
- Backend lookups by human-readable identifiers

## 2026-01-17 - Shop Name Exclusion Filter

This update adds the ability to exclude products from specific shops when searching. The new `excludeShopName` field allows users to filter out products from unwanted shops by exact shop name matching.

### Added

#### ProductSearchData Schema

A new `excludeShopName` field has been added:

**Field Structure**:
```json
{
  "language": "en",
  "currency": "USD",
  "productQuery": "vintage watch",
  "shopName": ["Sotheby's"],
  "excludeShopName": ["Heritage Auctions", "Christie's"]
}
```

**Field Details**:
- **Field name**: `excludeShopName`
- **Type**: `array of strings` (default: empty array)
- **Behavior**: Exact keyword match exclusion (case-sensitive)
- **Multiple values**: Supported (excludes products from any of the specified shops)
- **Default**: Empty array `[]` (no shops excluded)

**Technical Details**:
- The underlying OpenSearch query uses `must_not` clause with `terms` query
- Empty array `[]` means no shop name exclusion (includes all shops)
- Non-empty array excludes products whose shop name exactly matches one of the provided values
- Shop names are matched as complete strings, not partial matches
- Works in conjunction with `shopName` filter (include/exclude logic combined)

#### PatchProductSearchData Schema

The same `excludeShopName` field is available for partial updates:

**Field Structure**:
```json
{
  "excludeShopName": ["Bonhams", "Phillips"]
}
```

**Field Details**:
- **Field name**: `excludeShopName`
- **Type**: `array of strings` (optional, nullable)
- **Behavior**: Exact keyword match exclusion
- **When updating**: Providing `excludeShopName` array replaces the entire exclusion filter; omitting it leaves existing filter unchanged

#### API Endpoints Affected

**POST /api/v1/products/search**:
- Request body now accepts optional `excludeShopName` field
- Accepts array of exact shop names to exclude from search results
- Example request:
  ```json
  {
    "language": "de",
    "currency": "EUR",
    "productQuery": "classical music",
    "shopName": ["Sotheby's"],
    "excludeShopName": ["Heritage Auctions", "Christie's"],
    "price": {
      "min": 1000,
      "max": 50000
    }
  }
  ```

**POST /api/v1/search-filters**:
- Request body's `productSearch` object supports new `excludeShopName` field
- Shop name exclusion filter can be saved as part of user search filters

**PATCH /api/v1/search-filters/{userSearchFilterId}**:
- Request body's `productSearch` object supports new `excludeShopName` field
- Can update shop name exclusions independently of other filter criteria

**Response Behavior**:
- Products from excluded shops are completely filtered out of search results
- Total count reflects products after exclusion filter is applied
- Works correctly with pagination and sorting

## 2026-01-17 - Shop Name Filter: Text Search to Keyword Filter

This update changes the shop name filter from a fuzzy text search to an exact keyword match filter. The filter now accepts an array of exact shop names instead of a single text query string, enabling filtering by multiple specific shop names simultaneously.

### Changed

#### ProductSearchData Schema

The `shopNameQuery` field has been replaced with `shopName`:

**Before**:
```json
{
  "language": "en",
  "currency": "USD",
  "productQuery": "vintage watch",
  "shopNameQuery": "Christie"
}
```

**After**:
```json
{
  "language": "en",
  "currency": "USD",
  "productQuery": "vintage watch",
  "shopName": ["Christie's", "Sotheby's"]
}
```

**Field Changes**:
- **Field name**: `shopNameQuery` → `shopName`
- **Type**: `string` (optional, nullable) → `array of strings` (default: empty array)
- **Behavior**: Fuzzy text search with AUTO fuzziness → Exact keyword match (case-sensitive)
- **Multiple values**: Not supported → Supported (filters products from any of the specified shops)
- **Constraints**: Minimum 3 characters → No minimum length (accepts any valid shop name)

**Technical Details**:
- The underlying OpenSearch mapping for `shopName` changed from `text` type (analyzed, fuzzy-searchable) to `keyword` type (exact match)
- Empty array `[]` means no shop name filtering (matches all shops)
- Non-empty array filters to products whose shop name exactly matches one of the provided values
- Shop names are matched as complete strings, not partial matches

#### PatchProductSearchData Schema

The same change applies to the PATCH endpoint for updating search filters:

**Before**:
```json
{
  "shopNameQuery": "Updated Store"
}
```

**After**:
```json
{
  "shopName": ["Updated Store Name", "Another Store"]
}
```

**Field Changes**:
- **Field name**: `shopNameQuery` → `shopName`
- **Type**: `string` (optional, nullable) → `array of strings` (optional, nullable)
- **Behavior**: Fuzzy text search → Exact keyword match
- **When updating**: Providing `shopName` array replaces the entire filter; omitting it leaves existing filter unchanged

#### API Endpoints Affected

**POST /api/v1/products/search**:
- Request body field changed from `shopNameQuery` to `shopName`
- Now accepts array of exact shop names instead of fuzzy search string
- Example request:
  ```json
  {
    "language": "de",
    "currency": "EUR",
    "productQuery": "classical music",
    "shopName": ["Heritage Auctions", "Sotheby's", "Christie's"],
    "price": {
      "min": 1000,
      "max": 50000
    }
  }
  ```

**POST /api/v1/search-filters**:
- Request body's `productSearch` object uses new `shopName` field
- Shop name filter now accepts exact shop names as array

**PATCH /api/v1/search-filters/{userSearchFilterId}**:
- Request body's `productSearch` object uses new `shopName` field
- Update filter by providing array of exact shop names

**GET /api/v1/search-filters** and **GET /api/v1/search-filters/{userSearchFilterId}**:
- Response's `productSearch` object now contains `shopName` array instead of `shopNameQuery` string

### Migration Guide

**For Frontend Developers**:

1. **Update request payloads**: Change `shopNameQuery` to `shopName` and use array format:
   ```javascript
   // Before
   const searchRequest = {
     language: "en",
     currency: "USD",
     productQuery: "antique",
     shopNameQuery: "Christie"  // ❌ Old format
   };
   
   // After
   const searchRequest = {
     language: "en",
     currency: "USD",
     productQuery: "antique",
     shopName: ["Christie's"]  // ✅ New format - exact match
   };
   ```

2. **Handle multiple shops**: You can now filter by multiple shop names:
   ```javascript
   const searchRequest = {
     language: "en",
     currency: "USD",
     productQuery: "painting",
     shopName: ["Sotheby's", "Christie's", "Heritage Auctions"]
   };
   ```

3. **Use exact names**: The shop name must match exactly (case-sensitive). Partial matches no longer work:
   ```javascript
   // ❌ Will not match "Christie's"
   shopName: ["Christie"]
   
   // ✅ Will match "Christie's"
   shopName: ["Christie's"]
   ```

4. **Empty array for no filter**: Use empty array instead of null/undefined:
   ```javascript
   // Before (no shop filter)
   shopNameQuery: null
   
   // After (no shop filter)
   shopName: []  // or omit the field entirely
   ```

5. **Update response parsing**: When reading saved search filters, expect `shopName` array:
   ```javascript
   // Before
   const shopFilter = filter.productSearch.shopNameQuery;  // string or null
   
   // After
   const shopFilters = filter.productSearch.shopName;  // array of strings
   ```

### Removed

- **shopNameQuery** field (replaced by **shopName** in ProductSearchData and PatchProductSearchData schemas)
- Fuzzy text search capability for shop names (now requires exact match)
- Single shop name filtering (replaced by array-based filtering supporting multiple shops)

## 2026-01-15 - Product Price Estimates

This update adds price estimate fields to products, allowing antique dealers and auction houses to display estimated price ranges alongside actual selling prices. This is particularly useful for auction items where the final price may differ from the estimated value.

### Added

#### Product Price Estimate Fields

**GetProductData** (extended):
- **priceEstimateMin** (PriceData, optional): Minimum estimated price for the product
  - Type: PriceData object with currency and amount in minor units
  - Format: Same as the `price` field (e.g., `{"currency": "EUR", "amount": 2500}`)
  - Nullable: Yes
  - Only present when the seller provides an estimated minimum price
  - Common for auction items and antiques where valuation ranges are typical
  
  **Example**:
  ```json
  {
    "productId": "550e8400-e29b-41d4-a716-446655440000",
    "price": {
      "currency": "EUR",
      "amount": 3000
    },
    "priceEstimateMin": {
      "currency": "EUR",
      "amount": 2500
    },
    "priceEstimateMax": {
      "currency": "EUR",
      "amount": 3500
    }
  }
  ```

- **priceEstimateMax** (PriceData, optional): Maximum estimated price for the product
  - Type: PriceData object with currency and amount in minor units
  - Format: Same as the `price` field (e.g., `{"currency": "EUR", "amount": 3500}`)
  - Nullable: Yes
  - Only present when the seller provides an estimated maximum price
  - Typically used together with `priceEstimateMin` to indicate a value range

**PutProductData** (extended):
- **priceEstimateMin** (PriceData, optional): Minimum estimated price for creating/updating products
  - Type: PriceData object with currency and amount in minor units
  - Format: `{"currency": "EUR", "amount": 2500}`
  - Nullable: Yes
  - Optional field for creating or updating products
  - Sellers can provide price estimate ranges when listing items
  
  **Example**:
  ```json
  {
    "shopsProductId": "antique-vase-123",
    "title": {
      "text": "Victorian Vase",
      "language": "en"
    },
    "state": "AVAILABLE",
    "url": "https://antique-shop.com/item/123",
    "price": {
      "currency": "EUR",
      "amount": 1200
    },
    "priceEstimateMin": {
      "currency": "EUR",
      "amount": 1000
    },
    "priceEstimateMax": {
      "currency": "EUR",
      "amount": 1500
    }
  }
  ```

- **priceEstimateMax** (PriceData, optional): Maximum estimated price for creating/updating products
  - Type: PriceData object with currency and amount in minor units
  - Format: `{"currency": "EUR", "amount": 3500}`
  - Nullable: Yes
  - Optional field for creating or updating products
  - Used to specify the upper bound of the estimated value range

#### API Endpoints Affected

**GET /api/v1/products/{shopId}/{shopsProductId}**:
- Response now includes `priceEstimateMin` and `priceEstimateMax` fields when present
- Both fields are optional and may be null if no estimates are provided

**GET /api/v1/products/{shopId}/{shopsProductId}/similar**:
- Similar products now include `priceEstimateMin` and `priceEstimateMax` fields when available

**PUT /api/v1/products**:
- Request body can now include `priceEstimateMin` and `priceEstimateMax` fields
- Both fields are optional for each product in the items array

**POST /api/v1/products/search**:
- Search results now include `priceEstimateMin` and `priceEstimateMax` for each product when available

**GET /api/v1/me/watchlist**:
- Watchlist products now display `priceEstimateMin` and `priceEstimateMax` when present

#### Product History Events

**ProductCreatedEventPayloadData** (extended):
- Now includes `priceEstimateMin` and `priceEstimateMax` fields in the CREATED event payload
- These fields capture the initial price estimates when a product is first created
- Both fields are optional and use the PriceData type
- Helps track the complete initial state of a product including estimated values

### Usage Examples

#### Product with price estimates
GET /api/v1/products/{shopId}/{shopsProductId}
```json

{
  "item": {
    "productId": "550e8400-e29b-41d4-a716-446655440000",
    "shopName": "Antique Auctions Ltd",
    "title": {
      "text": "18th Century Mahogany Desk",
      "language": "en"
    },
    "price": {
      "currency": "GBP",
      "amount": 450000
    },
    "priceEstimateMin": {
      "currency": "GBP",
      "amount": 400000
    },
    "priceEstimateMax": {
      "currency": "GBP",
      "amount": 550000
    },
    "state": "AVAILABLE"
  }
}
```

#### Creating a product with price estimates
PUT /api/v1/products
```json
{
  "items": [
    {
      "shopsProductId": "auction-item-456",
      "title": {
        "text": "Rare Ming Dynasty Vase",
        "language": "en"
      },
      "state": "LISTED",
      "url": "https://auction-house.com/item/456",
      "priceEstimateMin": {
        "currency": "USD",
        "amount": 500000
      },
      "priceEstimateMax": {
        "currency": "USD",
        "amount": 800000
      }
    }
  }
}
```

### Implementation Notes

**Price Estimate Behavior**:
- Both `priceEstimateMin` and `priceEstimateMax` are completely independent from the actual `price` field
- Estimates can be present even when `price` is null (e.g., for auction items without a fixed price)
- The actual `price` may fall outside the estimate range (e.g., if a sold price exceeded expectations)
- Both estimate fields are optional and can be present independently
- Currency conversion is handled automatically by the enrichment service, same as for the `price` field

**Common Use Cases**:
1. **Auction Houses**: Display estimated value ranges before bidding
2. **Antique Dealers**: Show professional appraisal values alongside asking prices
3. **Collectibles**: Indicate market value estimates for rare items
4. **Insurance Purposes**: Provide valuation ranges for high-value antiques

**Frontend Impact**:
- Frontend applications should display price estimates when available
- Consider showing estimates differently from actual prices (e.g., "Estimated: £4,000 - £5,500" vs "Price: £4,750")
- For auction items without a fixed price, estimates are the primary pricing information
- Estimates are optional - handle cases where they are null or absent

**Backend Type**: The Rust backend uses:
- `native_price_estimate_min: Option<Price>` - Native currency minimum estimate
- `native_price_estimate_max: Option<Price>` - Native currency maximum estimate
- `other_price_estimate_min: HashMap<Currency, MonetaryAmount>` - Converted estimates for other currencies
- `other_price_estimate_max: HashMap<Currency, MonetaryAmount>` - Converted estimates for other currencies

### Changed

No existing fields or endpoints were modified. This is a purely additive change.

### Removed

No endpoints, fields, or functionality have been removed in this update.

## 2026-01-14 - Auction DateTime Fields

This update adds auction timing information for products, enabling users to see when items will be auctioned and filter searches by auction timing. Auction houses typically list time windows for when items will be sold, and this feature makes that information available through the API.

### Added

#### Product Fields

**GetProductData** (extended):
- **auctionStart** (string, date-time, optional): Start datetime of the auction window
  - Type: ISO8601/RFC3339 datetime string
  - Format: `date-time` (e.g., `"2026-02-15T10:00:00Z"`)
  - Nullable: Yes
  - Only present for products from auction houses with scheduled auction times
  - Indicates when bidding begins or when the item will be auctioned
  
  **Example**:
  ```json
  {
    "productId": "550e8400-e29b-41d4-a716-446655440000",
    "auctionStart": "2026-02-15T10:00:00Z",
    "auctionEnd": "2026-02-15T14:00:00Z"
  }
  ```

- **auctionEnd** (string, date-time, optional): End datetime of the auction window
  - Type: ISO8601/RFC3339 datetime string
  - Format: `date-time` (e.g., `"2026-02-15T14:00:00Z"`)
  - Nullable: Yes
  - Only present for products from auction houses with scheduled auction times
  - Indicates when bidding ends or when the auction session concludes

**PutProductData** (extended):
- **auctionStart** (string, date-time, optional): Start datetime of the auction window
  - Type: ISO8601/RFC3339 datetime string
  - Format: `date-time`
  - Nullable: Yes
  - Optional field for creating or updating products
  - Only applicable for products from auction houses
  
  **Example**:
  ```json
  {
    "shopsProductId": "auction-item-123",
    "title": {
      "text": "Antique Vase",
      "language": "en"
    },
    "state": "LISTED",
    "url": "https://auction-house.com/item/123",
    "auctionStart": "2026-02-15T10:00:00Z",
    "auctionEnd": "2026-02-15T14:00:00Z"
  }
  ```

- **auctionEnd** (string, date-time, optional): End datetime of the auction window
  - Type: ISO8601/RFC3339 datetime string
  - Format: `date-time`
  - Nullable: Yes
  - Optional field for creating or updating products
  - Only applicable for products from auction houses

#### Search Filter Fields

**ProductSearchData** (extended):
- **auctionStart** (RangeQueryDateTime, optional): Filter by auction start datetime range
  - Type: Object with `min` and/or `max` ISO8601/RFC3339 datetime strings
  - Nullable: Yes
  - Filters products by when their auction windows begin
  - Only matches products that have auction start times set
  - Both `min` and `max` are optional within the range
  
  **Example**:
  ```json
  {
    "language": "de",
    "currency": "EUR",
    "productQuery": "antique furniture",
    "auctionStart": {
      "min": "2026-01-01T00:00:00Z",
      "max": "2026-03-31T23:59:59Z"
    }
  }
  ```

- **auctionEnd** (RangeQueryDateTime, optional): Filter by auction end datetime range
  - Type: Object with `min` and/or `max` ISO8601/RFC3339 datetime strings
  - Nullable: Yes
  - Filters products by when their auction windows end
  - Only matches products that have auction end times set
  - Both `min` and `max` are optional within the range
  
  **Example**:
  ```json
  {
    "language": "en",
    "currency": "USD",
    "productQuery": "vintage watch",
    "auctionEnd": {
      "max": "2026-02-28T23:59:59Z"
    }
  }
  ```

**PatchProductSearchData** (extended):
- **auctionStart** (RangeQueryDateTime, optional): Filter by auction start datetime range
  - Type: Object with `min` and/or `max` ISO8601/RFC3339 datetime strings
  - Nullable: Yes
  - Can be updated independently when patching a search filter
  - Filters products by when their auction windows begin
  - Only matches products that have auction start times set

- **auctionEnd** (RangeQueryDateTime, optional): Filter by auction end datetime range
  - Type: Object with `min` and/or `max` ISO8601/RFC3339 datetime strings
  - Nullable: Yes
  - Can be updated independently when patching a search filter
  - Filters products by when their auction windows end
  - Only matches products that have auction end times set

#### API Endpoints Affected

**GET /api/v1/products/{shopId}/{shopsProductId}**:
- Response now includes `auctionStart` and `auctionEnd` fields when present

**PUT /api/v1/products**:
- Request body can now include `auctionStart` and `auctionEnd` fields

**POST /api/v1/products/search**:
- Request body can now include `auctionStart` and `auctionEnd` query filters

**POST /api/v1/users/{userId}/search-filters**:
- Can create search filters with `auctionStart` and `auctionEnd` query filters

**PATCH /api/v1/users/{userId}/search-filters/{searchFilterId}**:
- Can update search filters with `auctionStart` and `auctionEnd` query filters

**GET /api/v1/users/{userId}/search-filters**:
- Returns search filters that may include `auctionStart` and `auctionEnd` query filters

**GET /api/v1/users/{userId}/search-filters/{searchFilterId}**:
- Returns search filter that may include `auctionStart` and `auctionEnd` query filters

### Usage Examples

#### Filtering products by auction timing

Find products being auctioned in February 2026:
```json
POST /api/v1/products/search
{
  "language": "de",
  "currency": "EUR",
  "productQuery": "antique",
  "auctionStart": {
    "min": "2026-02-01T00:00:00Z",
    "max": "2026-02-28T23:59:59Z"
  }
}
```

Find products with auctions ending before a specific date:
```json
POST /api/v1/products/search
{
  "language": "en",
  "currency": "GBP",
  "productQuery": "vintage",
  "auctionEnd": {
    "max": "2026-01-31T23:59:59Z"
  }
}
```

Find products with auctions starting after a specific date:
```json
POST /api/v1/products/search
{
  "language": "fr",
  "currency": "EUR",
  "productQuery": "furniture",
  "auctionStart": {
    "min": "2026-03-01T00:00:00Z"
  }
}
```

## 2026-01-14 - Shop Type Classification

This update introduces shop type classification to distinguish between different types of vendors (auction houses, commercial dealers, and marketplaces), enabling users to filter products and shops by vendor type.

### Added

#### New Data Types

**ShopTypeData** (enum):
- Classification of shop/vendor type
- Possible values:
  - `AUCTION_HOUSE`: Auction house selling items through auctions
  - `COMMERCIAL_DEALER`: Commercial dealer or shop selling items directly
  - `MARKETPLACE`: Marketplace platform connecting buyers and sellers
- No default value (field is mandatory)
- Example: `"COMMERCIAL_DEALER"`

#### New Filter Fields

**ProductSearchData and PatchProductSearchData**:
- **shopType** (array of ShopTypeData, optional): Filter products by shop type
  - Type: Array of enum strings
  - Values: `AUCTION_HOUSE`, `COMMERCIAL_DEALER`, `MARKETPLACE`
  - Default: Empty array `[]`
  - Nullable: Yes
  - Unique items: Yes
  
  **Example**:
  ```json
  {
    "shopType": ["COMMERCIAL_DEALER", "AUCTION_HOUSE"]
  }
  ```

**ShopSearchData**:
- **shopType** (array of ShopTypeData, optional): Filter shops by shop type
  - Type: Array of enum strings
  - Values: `AUCTION_HOUSE`, `COMMERCIAL_DEALER`, `MARKETPLACE`
  - Default: Empty array `[]`
  - Nullable: Yes
  - Unique items: Yes
  
  **Example**:
  ```json
  {
    "shopType": ["MARKETPLACE"]
  }
  ```

### Changed

#### Shop Data Types

All shop-related data types now include the mandatory `shopType` field:

**GetShopData**:
- **Added field**: `shopType` (ShopTypeData, required): Type classification of the shop
- This field is now required in all shop responses

**Example**:
```json
{
  "shopId": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Tech Store Premium",
  "shopType": "COMMERCIAL_DEALER",
  "domains": ["tech-store-premium.com"],
  "created": "2024-01-01T10:00:00Z",
  "updated": "2024-01-01T12:00:00Z"
}
```

**PostShopData**:
- **Added field**: `shopType` (ShopTypeData, required): Type classification of the shop
- This field is now mandatory when creating a new shop

**Example**:
```json
{
  "name": "Tech Store Premium",
  "shopType": "COMMERCIAL_DEALER",
  "domains": ["tech-store-premium.com"]
}
```

**PatchShopData**:
- **Added field**: `shopType` (ShopTypeData, optional): New type classification for the shop
- This field is optional and can be updated independently

**Example**:
```json
{
  "shopType": "AUCTION_HOUSE"
}
```

#### Product Data Types

**GetProductData**:
- **Added field**: `shopType` (ShopTypeData, required): Type of the shop selling this product
- This field is now included in all product responses, providing shop type information at the product level

**Example**:
```json
{
  "productId": "550e8400-e29b-41d4-a716-446655440000",
  "shopId": "660e8400-e29b-41d4-a716-446655440001",
  "shopName": "Antique Auctions Ltd",
  "shopType": "AUCTION_HOUSE",
  "title": {
    "text": "Victorian Writing Desk",
    "language": "en"
  }
}
```

#### Affected Endpoints

All endpoints returning shop or product data now include the `shopType` field:

**Shop Endpoints**:
- `POST /api/v1/shops` - Now requires `shopType` in request body
- `GET /api/v1/shops/{shopIdentifier}` - Returns `shopType` in response
- `PATCH /api/v1/shops/{shopIdentifier}` - Accepts optional `shopType` in request body, returns `shopType` in response
- `POST /api/v1/shops/search` - Returns `shopType` for each shop, accepts `shopType` filter in request body

**Product Endpoints**:
- `GET /api/v1/products/{shopId}/{shopsProductId}` - Returns `shopType` in product data
- `POST /api/v1/products/search` - Returns `shopType` for each product, accepts `shopType` filter in request body
- `GET /api/v1/products/{shopId}/{shopsProductId}/similar` - Returns `shopType` for each similar product
- `GET /api/v1/me/watchlist` - Returns `shopType` for each watchlist product

**Search Filter Endpoints**:
- `POST /api/v1/me/search-filters` - Accepts `shopType` in `productSearch` criteria
- `GET /api/v1/me/search-filters` - Returns `shopType` in saved filter criteria
- `GET /api/v1/me/search-filters/{userSearchFilterId}` - Returns `shopType` in filter criteria
- `PATCH /api/v1/me/search-filters/{userSearchFilterId}` - Accepts `shopType` in partial `productSearch` update

#### Search Behavior

- The `shopType` filter works as a disjunctive (OR) filter: products/shops matching ANY of the specified types will be returned
- Empty arrays or null values for the `shopType` filter will not apply any filtering for this criterion
- The filter combines with other search criteria using AND logic

**Example Product Search Request**:
```json
POST /api/v1/products/search
{
  "language": "en",
  "currency": "USD",
  "productQuery": "antique furniture",
  "shopType": ["AUCTION_HOUSE", "COMMERCIAL_DEALER"]
}
```

This request will return products from shops that are either auction houses OR commercial dealers, matching the search query "antique furniture".

**Example Shop Search Request**:
```json
POST /api/v1/shops/search
{
  "shopNameQuery": "antique",
  "shopType": ["AUCTION_HOUSE"]
}
```

This request will return shops that match "antique" in their name AND are auction houses.

### Frontend Impact

**Breaking Changes**:
1. **Shop Creation**: Frontend must now provide `shopType` when creating shops via `POST /api/v1/shops`
2. **Data Models**: Update TypeScript/data models to include the mandatory `shopType` field in:
   - `GetShopData`
   - `PostShopData`
   - `GetProductData`
3. **Optional Fields**: Update models to include optional `shopType` field in:
   - `PatchShopData`
   - `ProductSearchData`
   - `PatchProductSearchData`
   - `ShopSearchData`

**New Features**:
1. **Shop Type Display**: Display shop type badges or indicators in:
   - Shop listings and search results
   - Product details (showing the type of shop selling the product)
   - Shop detail pages
2. **Filtering**: Implement UI controls to filter by shop type in:
   - Product search interfaces
   - Shop search interfaces
   - Saved search filter creation/editing
3. **Shop Management**: Add shop type selection in shop creation and editing forms

**Example TypeScript Updates**:
```typescript
// Before
interface GetShopData {
  shopId: string;
  name: string;
  domains: string[];
  image?: string;
  created: string;
  updated: string;
}

// After
interface GetShopData {
  shopId: string;
  name: string;
  shopType: 'AUCTION_HOUSE' | 'COMMERCIAL_DEALER' | 'MARKETPLACE';
  domains: string[];
  image?: string;
  created: string;
  updated: string;
}

// Product search filter
interface ProductSearchData {
  language: string;
  currency: string;
  productQuery: string;
  shopNameQuery?: string;
  shopType?: ('AUCTION_HOUSE' | 'COMMERCIAL_DEALER' | 'MARKETPLACE')[];
  // ... other fields
}
```

### Removed

No endpoints, fields, or functionality have been removed in this update. All changes are additive, though the `shopType` field is mandatory in certain contexts.

## 2026-01-11 - Product Image Prohibited Content Classification

This update enhances product images with prohibited content classification to identify and flag images containing sensitive or prohibited content such as Nazi Germany symbols and insignia.

### Added

#### New Data Types

**ProhibitedContentData** (enum):
- Classification of prohibited or sensitive content that may be present in product images
- Possible values:
  - `UNKNOWN`: Content classification has not been determined
  - `NONE`: No prohibited content detected in the image
  - `NAZI_GERMANY`: Image contains Nazi Germany symbols, insignia, or related content
- Default value: `UNKNOWN`

**ProductImageData** (object):
- Product image with prohibited content classification
- Properties:
  - `url` (string, required): URL to the product image
  - `prohibitedContent` (ProhibitedContentData, required): Prohibited content classification for this image
- Example:
  ```json
  {
    "url": "https://my-shop.com/images/product-1.jpg",
    "prohibitedContent": "NONE"
  }
  ```

### Changed

#### GET /api/v1/products/{shopId}/{shopsProductId}

**Response Structure Change**: The `images` field in the product response has been enhanced from a simple array of URL strings to an array of `ProductImageData` objects that include prohibited content classification.

**Endpoint**: `GET /api/v1/products/{shopId}/{shopsProductId}`

**Old Response Structure**:
```json
{
  "item": {
    "productId": "550e8400-e29b-41d4-a716-446655440000",
    "images": [
      "https://my-shop.com/images/product-1.jpg",
      "https://my-shop.com/images/product-2.jpg"
    ]
  }
}
```

**New Response Structure**:
```json
{
  "item": {
    "productId": "550e8400-e29b-41d4-a716-446655440000",
    "images": [
      {
        "url": "https://my-shop.com/images/product-1.jpg",
        "prohibitedContent": "NONE"
      },
      {
        "url": "https://my-shop.com/images/product-2.jpg",
        "prohibitedContent": "NAZI_GERMANY"
      }
    ]
  }
}
```

**Impact on Other Endpoints**:
This change affects all endpoints that return product data with images:
- `GET /api/v1/products/search` - Product search results
- `GET /api/v1/products/similar/{shopId}/{shopsProductId}` - Similar products
- `GET /api/v1/me/watchlist` - Watchlist products

**Frontend Impact**:
- Frontend applications must update their code to access image URLs via `image.url` instead of using the URL string directly
- Display appropriate warnings or content filters based on the `prohibitedContent` classification
- Handle the `NAZI_GERMANY` classification according to regional legal requirements (e.g., Germany's restrictions on Nazi symbols)
- The `prohibitedContent` field may be `UNKNOWN` during a transition period while existing images are being classified

**Note**: This is a breaking change to the API response structure. Frontend applications must be updated to handle the new image object structure.

## 2026-01-06 - Watchlist Quota Restriction

This update implements a quota limit on the number of products a user can add to their watchlist, preventing unlimited watchlist growth and ensuring fair resource usage.

### Added

#### New Error Code

**WATCHLIST_QUOTA_EXCEEDED**:
- Error code returned when a user attempts to add a product to their watchlist after reaching the maximum quota
- Used in the watchlist product creation endpoint

#### New Error Response for POST /api/v1/me/watchlist

**422 Unprocessable Entity** - Watchlist quota exceeded:

When a user attempts to add a product to their watchlist but already has the maximum number of products (5), the API now returns:

- **Status Code**: 422 Unprocessable Entity
- **Response Body**:
  ```json
  {
    "status": 422,
    "title": "Unprocessable Content",
    "error": "WATCHLIST_QUOTA_EXCEEDED",
    "detail": "Exceeded the maximum amount of watchlist entries. There are already 5/5 watchlist entries occupied."
  }
  ```

### Changed

#### POST /api/v1/me/watchlist

**Behavior Change**: The endpoint now enforces a quota limit of 5 watchlist products per user.

- **Endpoint**: `POST /api/v1/me/watchlist`
- **Maximum Quota**: 5 watchlist products per user
- **Validation**: Before creating a new watchlist entry, the system checks if the user already has 5 or more products in their watchlist
- **New Response**: Returns HTTP 422 with error code `WATCHLIST_QUOTA_EXCEEDED` when quota is exceeded

**Updated Description**: The endpoint description now includes information about the 5-product quota limit:
> "Each user is limited to a maximum of 5 watchlist products. If the user already has 5 products in their watchlist, adding another will result in a 422 Unprocessable Entity error."

**Request**: No changes to request format
```json
{
  "shopId": "550e8400-e29b-41d4-a716-446655440000",
  "shopsProductId": "6ba7b810"
}
```

**Updated Responses**:
- **201 Created**: Product successfully added to watchlist (when quota not exceeded)
- **400 Bad Request**: Invalid request body
- **401 Unauthorized**: Invalid or missing JWT token
- **422 Unprocessable Entity**: Watchlist quota exceeded (NEW)
- **500 Internal Server Error**: Internal server error

**Frontend Impact**:
- Frontend applications should handle the new 422 error response
- Display appropriate user feedback when the watchlist quota is reached
- Consider showing the user's current watchlist count (e.g., "5/5 products in watchlist")
- Provide options to remove existing products before adding new ones

## 2026-01-04 - Product Search Filter Enhancement

This update extends the product search filtering capabilities by adding new filter options for origin year and product attributes (authenticity, condition, provenance, restoration).

### Added

#### New Search Filter Fields

The `ProductSearchData` and `PatchProductSearchData` objects now support filtering by additional product attributes:

**New Fields in `ProductSearchData`**:

1. **originYear** (RangeQueryInt32, optional): Filter products by origin year range
   - Type: Object with optional `min` and `max` integer fields
   - Description: Filter products whose origin year falls within the specified range
   - Both `min` and `max` are optional and inclusive
   - Nullable: Yes
   
   **Example**:
   ```json
   {
     "originYear": {
       "min": 1742,
       "max": 1953
     }
   }
   ```

2. **authenticity** (Array of AuthenticityData, optional): Filter by authenticity classifications
   - Type: Array of enum strings
   - Values: `ORIGINAL`, `LATER_COPY`, `REPRODUCTION`, `QUESTIONABLE`, `UNKNOWN`
   - Default: Empty array `[]`
   - Nullable: Yes
   - Unique items: Yes
   
   **Example**:
   ```json
   {
     "authenticity": ["ORIGINAL", "LATER_COPY"]
   }
   ```

3. **condition** (Array of ConditionData, optional): Filter by product condition assessments
   - Type: Array of enum strings
   - Values: `EXCELLENT`, `GREAT`, `GOOD`, `FAIR`, `POOR`, `UNKNOWN`
   - Default: Empty array `[]`
   - Nullable: Yes
   - Unique items: Yes
   
   **Example**:
   ```json
   {
     "condition": ["EXCELLENT", "GREAT"]
   }
   ```

4. **provenance** (Array of ProvenanceData, optional): Filter by provenance documentation levels
   - Type: Array of enum strings
   - Values: `COMPLETE`, `PARTIAL`, `CLAIMED`, `NONE`, `UNKNOWN`
   - Default: Empty array `[]`
   - Nullable: Yes
   - Unique items: Yes
   
   **Example**:
   ```json
   {
     "provenance": ["PARTIAL", "COMPLETE"]
   }
   ```

5. **restoration** (Array of RestorationData, optional): Filter by restoration work levels
   - Type: Array of enum strings
   - Values: `NONE`, `MINOR`, `MAJOR`, `UNKNOWN`
   - Default: Empty array `[]`
   - Nullable: Yes
   - Unique items: Yes
   
   **Example**:
   ```json
   {
     "restoration": ["NONE", "MINOR"]
   }
   ```

#### New Sort Field

**SortProductFieldData** enum now includes:

- **originYear**: Sort products by their origin year (ascending or descending)
  - Can be used with the `sort` query parameter on search endpoints
  - Supports both `asc` and `desc` ordering via the `order` parameter

**Updated Enum Values**: `score`, `price`, `originYear`, `updated`, `created`

#### New Schema Type

**RangeQueryInt32**:
- Type: Object for integer range queries
- Properties:
  - `min` (integer, optional): Minimum value (inclusive)
  - `max` (integer, optional): Maximum value (inclusive)
- Used for filtering by origin year ranges

### Changed

#### Affected Endpoints

All endpoints that accept `ProductSearchData` now support the new filter fields:

1. **POST /api/v1/products/search**
   - Request body `ProductSearchData` now accepts `originYear`, `authenticity`, `condition`, `provenance`, and `restoration` fields
   - Can now sort results by `originYear` using the `sort` query parameter
   - All new filter fields are optional

2. **POST /api/v1/me/search-filters**
   - Request body `PostUserSearchFilterData.productSearch` now accepts the new filter fields
   - Users can save search filters with these additional criteria

3. **PATCH /api/v1/me/search-filters/{userSearchFilterId}**
   - Request body `PatchUserSearchFilterData.productSearch` now accepts the new filter fields
   - Users can update existing search filters to include these criteria

#### Search Behavior

- All new filter fields work as conjunctive (AND) filters when specified
- When array filter fields (authenticity, condition, provenance, restoration) contain multiple values, products matching ANY of the specified values will be returned (disjunctive OR within each field)
- Empty arrays or null values for these fields will not apply any filtering for that criterion
- The `originYear` range filter includes products whose origin year falls within the specified min/max bounds (inclusive)

**Example Request**:
```json
POST /api/v1/products/search
{
  "language": "en",
  "currency": "USD",
  "productQuery": "antique vase",
  "originYear": {
    "min": 1800,
    "max": 1900
  },
  "authenticity": ["ORIGINAL"],
  "condition": ["EXCELLENT", "GREAT"],
  "provenance": ["COMPLETE", "PARTIAL"]
}
```

This request will search for:
- Products matching "antique vase"
- With origin year between 1800-1900
- That are verified ORIGINAL
- In EXCELLENT or GREAT condition
- With COMPLETE or PARTIAL provenance

## 2026-01-01 - Product Attribute Enrichment

This update introduces comprehensive attribute enrichment for antique products, adding fields for origin year, authenticity, condition, provenance, and restoration status.

### Added

#### New Product Attribute Fields

All endpoints that return `GetProductData` now include the following optional fields with detailed antique product attributes:

**New Fields in `GetProductData`**:

1. **Origin Year Fields** - Temporal information about when the antique was created:
   - `originYearMin` (integer, optional): Lower bound of estimated origin year range
   - `originYear` (integer, optional): Exact year of origin (when known precisely)
   - `originYearMax` (integer, optional): Upper bound of estimated origin year range
   
   **Rules**:
   - When `originYear` is present, both `originYearMin` and `originYearMax` will be `null`
   - When the year is expressed as a range, `originYear` will be `null` and min/max will contain the range bounds
   - Either min or max can be present alone to indicate "after this year" or "before this year"
   - All year fields are nullable and optional

   **Examples**:
   ```json
   // Exact year known
   {
     "originYear": 1837,
     "originYearMin": null,
     "originYearMax": null
   }
   
   // Year range estimated
   {
     "originYear": null,
     "originYearMin": 1900,
     "originYearMax": 1950
   }
   
   // Only upper bound known (before this year)
   {
     "originYear": null,
     "originYearMin": null,
     "originYearMax": 1800
   }
   
   // Only lower bound known (after this year)
   {
     "originYear": null,
     "originYearMin": 1700,
     "originYearMax": null
   }
   
   // No origin year information
   {
     "originYear": null,
     "originYearMin": null,
     "originYearMax": null
   }
   ```

2. **authenticity** (AuthenticityData, optional): Authenticity classification
   - Type: Enum string
   - Values: `ORIGINAL`, `LATER_COPY`, `REPRODUCTION`, `QUESTIONABLE`, `UNKNOWN`
   - Default: `UNKNOWN`
   - Nullable: Yes

3. **condition** (ConditionData, optional): Physical condition assessment
   - Type: Enum string
   - Values: `EXCELLENT`, `GREAT`, `GOOD`, `FAIR`, `POOR`, `UNKNOWN`
   - Default: `UNKNOWN`
   - Nullable: Yes

4. **provenance** (ProvenanceData, optional): Documentation trail and ownership history
   - Type: Enum string
   - Values: `COMPLETE`, `PARTIAL`, `CLAIMED`, `NONE`, `UNKNOWN`
   - Default: `UNKNOWN`
   - Nullable: Yes

5. **restoration** (RestorationData, optional): Level of restoration work performed
   - Type: Enum string
   - Values: `NONE`, `MINOR`, `MAJOR`, `UNKNOWN`
   - Default: `UNKNOWN`
   - Nullable: Yes

#### New Data Type Enums

**AuthenticityData**
- Authenticity classification of antique products
- Values and meanings:
  - `ORIGINAL`: Verified original antique from the stated period
  - `LATER_COPY`: Antique copy made at a later time but still historical
  - `REPRODUCTION`: Modern reproduction or replica
  - `QUESTIONABLE`: Authenticity is disputed or uncertain
  - `UNKNOWN`: Authenticity has not been determined (default)

**ConditionData**
- Physical condition assessment of antique products
- Values and meanings:
  - `EXCELLENT`: Near-perfect condition with minimal wear
  - `GREAT`: Very good condition with minor signs of age
  - `GOOD`: Good condition with moderate wear consistent with age
  - `FAIR`: Fair condition with significant wear but structurally sound
  - `POOR`: Poor condition with major damage or deterioration
  - `UNKNOWN`: Condition has not been assessed (default)

**ProvenanceData**
- Documentation trail and ownership history
- Values and meanings:
  - `COMPLETE`: Full documented history from origin to present
  - `PARTIAL`: Some documentation exists but history has gaps
  - `CLAIMED`: Provenance is claimed by seller but lacks documentation
  - `NONE`: No provenance documentation available
  - `UNKNOWN`: Provenance status has not been determined (default)

**RestorationData**
- Level of restoration work performed
- Values and meanings:
  - `NONE`: No restoration, original condition preserved
  - `MINOR`: Minor restoration or conservation work (cleaning, small repairs)
  - `MAJOR`: Significant restoration or reconstruction work
  - `UNKNOWN`: Restoration history has not been determined (default)

### Changed

#### Affected Endpoints

All endpoints returning `GetProductData` (either directly or wrapped in `PersonalizedGetProductData`) now include these new optional fields:

1. **GET /api/v1/products/{shopId}/{shopsProductId}**
   - Response body now includes origin year and attribute fields
   - All new fields are optional and may be `null`

2. **GET /api/v1/products/{shopId}/{shopsProductId}/similar**
   - Each similar product in the response array includes the new attribute fields

3. **POST /api/v1/products/search**
   - Search results now include attribute information for each product

4. **GET /api/v1/me/watchlist**
   - Watchlist products now display enriched attribute information

### Example Product Responses

**Complete Example with All New Fields**:
```json
{
  "item": {
    "productId": "550e8400-e29b-41d4-a716-446655440000",
    "eventId": "550e8400-e29b-41d4-a716-446655440001",
    "shopId": "550e8400-e29b-41d4-a716-446655440000",
    "shopsProductId": "chopin-etudes-1833",
    "shopName": "Historic Manuscripts Ltd",
    "title": {
      "text": "Frédéric Chopin - Études Op. 10 - First Edition 1833",
      "language": "en"
    },
    "description": {
      "text": "Rare first edition of Chopin's groundbreaking Études Op. 10...",
      "language": "en"
    },
    "price": {
      "currency": "EUR",
      "amount": 450000
    },
    "state": "AVAILABLE",
    "url": "https://historic-manuscripts.com/chopin-etudes-op10-1833",
    "images": [
      "https://historic-manuscripts.com/images/chopin-1.jpg"
    ],
    "originYear": 1833,
    "originYearMin": null,
    "originYearMax": null,
    "authenticity": "ORIGINAL",
    "condition": "EXCELLENT",
    "provenance": "COMPLETE",
    "restoration": "MINOR",
    "created": "2024-01-01T10:00:00Z",
    "updated": "2024-01-01T12:00:00Z"
  },
  "userState": {
    "watchlist": {
      "watching": true,
      "notifications": true
    }
  }
}
```

**Example with Year Range**:
```json
{
  "productId": "660e8400-e29b-41d4-a716-446655440001",
  "shopName": "Antique Furniture Gallery",
  "title": {
    "text": "Victorian Mahogany Writing Desk",
    "language": "en"
  },
  "originYear": null,
  "originYearMin": 1837,
  "originYearMax": 1901,
  "authenticity": "QUESTIONABLE",
  "condition": "GOOD",
  "provenance": "PARTIAL",
  "restoration": "MAJOR"
}
```

**Example with Only Upper Bound**:
```json
{
  "productId": "880e8400-e29b-41d4-a716-446655440003",
  "shopName": "Ancient Artifacts Museum",
  "title": {
    "text": "Roman Bronze Lamp",
    "language": "en"
  },
  "originYear": null,
  "originYearMin": null,
  "originYearMax": 300,
  "authenticity": "ORIGINAL",
  "condition": "FAIR",
  "provenance": "PARTIAL"
}
```

**Example with Minimal Attributes**:
```json
{
  "productId": "770e8400-e29b-41d4-a716-446655440002",
  "shopName": "Modern Reproductions Inc",
  "title": {
    "text": "Replica Louis XVI Chair",
    "language": "en"
  },
  "originYear": 2020,
  "authenticity": "REPRODUCTION",
  "condition": "EXCELLENT",
  "provenance": "NONE",
  "restoration": "NONE"
}
```

### Removed

No endpoints, fields, or functionality have been removed in this update. All changes are additive and backward compatible.

## 2025-12-16 - Shop API Identifier Flexibility

This update enhances the shop API endpoints to accept both shop IDs (UUIDs) and shop domains as identifiers, providing more flexible ways to retrieve and update shop information. This change allows clients to reference shops using either their unique UUID or any of their registered domains.

### Changed

#### GET /api/v1/shops/{shopIdentifier}

The endpoint path parameter has been changed from `{shopId}` to `{shopIdentifier}` to support multiple identifier formats.

**Path Changes**:
- **Before**: `/api/v1/shops/{shopId}`
- **After**: `/api/v1/shops/{shopIdentifier}`

**Parameter Changes**:

Before (UUID only):
```
Parameter: shopId (required)
Type: string (UUID format)
Example: "550e8400-e29b-41d4-a716-446655440000"
```

After (UUID or domain):
```
Parameter: shopIdentifier (required)
Type: string
Accepts:
  - Shop ID (UUID format): "550e8400-e29b-41d4-a716-446655440000"
  - Shop domain: "tech-store-premium.com", "shop.example.com"
```

**Description Updated**:
The endpoint now explicitly documents that it accepts both:
1. **Shop ID (UUID)**: The unique identifier assigned to the shop when created
2. **Shop Domain**: Any domain registered to the shop (e.g., "tech-store.com", "example.shop.com")

**Examples**:

Using shop ID:
```http
GET /api/v1/shops/550e8400-e29b-41d4-a716-446655440000
```

Using shop domain:
```http
GET /api/v1/shops/tech-store-premium.com
```

**Error Response Changes**:

**400 Bad Request** error examples updated:

Before:
- `BAD_PATH_PARAMETER_VALUE`: "Missing field 'shopId'"
- `INVALID_UUID`: "Invalid UUID format" (for field "shopId")

After:
- `BAD_PATH_PARAMETER_VALUE`: "Missing field 'shopIdentifier'"
- `INVALID_SHOP_IDENTIFIER`: "Invalid shop identifier format" (for field "shopIdentifier")

**Migration Impact**:
- Frontend must update API calls from `/api/v1/shops/{shopId}` to `/api/v1/shops/{shopIdentifier}`
- Both UUID and domain string formats are now accepted
- Domain format provides a more intuitive way to reference shops
- Existing UUID-based calls continue to work with the new parameter name

#### PATCH /api/v1/shops/{shopIdentifier}

The endpoint path parameter has been changed from `{shopId}` to `{shopIdentifier}` to support multiple identifier formats.

**Path Changes**:
- **Before**: `/api/v1/shops/{shopId}`
- **After**: `/api/v1/shops/{shopIdentifier}`

**Parameter Changes**:

Before (UUID only):
```
Parameter: shopId (required)
Type: string (UUID format)
Example: "550e8400-e29b-41d4-a716-446655440000"
```

After (UUID or domain):
```
Parameter: shopIdentifier (required)
Type: string
Accepts:
  - Shop ID (UUID format): "550e8400-e29b-41d4-a716-446655440000"
  - Shop domain: "tech-store-premium.com", "shop.example.com"
```

**Description Updated**:
The endpoint now explicitly documents that it accepts both shop ID (UUID) and shop domain as the identifier.

**Examples**:

Using shop ID:
```http
PATCH /api/v1/shops/550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{
  "name": "Updated Shop Name"
}
```

Using shop domain:
```http
PATCH /api/v1/shops/tech-store-premium.com
Content-Type: application/json

{
  "name": "Updated Shop Name"
}
```

**Error Response Changes**:

**400 Bad Request** error examples updated:

Before:
- `BAD_PATH_PARAMETER_VALUE`: "Missing field 'shopId'"
- `INVALID_UUID`: "Invalid UUID format" (for field "shopId")

After:
- `BAD_PATH_PARAMETER_VALUE`: "Missing field 'shopIdentifier'"
- `INVALID_SHOP_IDENTIFIER`: "Invalid shop identifier format" (for field "shopIdentifier")

**Migration Impact**:
- Frontend must update API calls from `/api/v1/shops/{shopId}` to `/api/v1/shops/{shopIdentifier}`
- Both UUID and domain string formats are now accepted
- Domain format allows updating a shop using any of its registered domains
- Request and response body structures remain unchanged

### Added

#### New Error Code: INVALID_SHOP_IDENTIFIER

**Error Code**: `INVALID_SHOP_IDENTIFIER`
- **HTTP Status**: 400 Bad Request
- **Endpoints**: `GET /api/v1/shops/{shopIdentifier}`, `PATCH /api/v1/shops/{shopIdentifier}`
- **When**: Provided shop identifier is neither a valid UUID nor a valid domain format
- **Description**: The shop identifier could not be parsed as either a UUID or a domain

**Error Response Example**:
```json
{
  "status": 400,
  "title": "Bad Request",
  "error": "INVALID_SHOP_IDENTIFIER",
  "source": {
    "field": "shopIdentifier",
    "sourceType": "path"
  },
  "detail": "Invalid shop identifier format"
}
```

**Examples of Invalid Shop Identifiers**:
- Empty string: `""`
- Invalid UUID: `"not-a-uuid"`
- Invalid domain: `"localhost"` (no TLD)
- Special characters: `"shop@example"`
- Malformed: `"..shop.com"`

**Frontend Impact**:
- Handle `INVALID_SHOP_IDENTIFIER` as a new 400 error type for shop endpoints
- Display appropriate error message when shop identifier is malformed
- Validate shop identifier format client-side before making requests

### Implementation Notes

**Shop Identifier Format**:
The `shopIdentifier` parameter accepts two formats:

1. **UUID Format** (Shop ID):
   - Standard UUID v4 format: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`
   - Example: `"550e8400-e29b-41d4-a716-446655440000"`
   - Case-insensitive
   
2. **Domain Format**:
   - Valid internet domain name
   - Must contain at least one dot (TLD required)
   - Examples: 
     - `"tech-store.com"`
     - `"shop.example.com"`
     - `"subdomain.shop.co.uk"`
   - Domain matching is case-insensitive
   - Must be a domain registered to the shop

**Backend Type**: New `ShopIdentifierData` enum in Rust
- Serializes/deserializes as a plain string (untagged enum)
- Automatically determines whether input is UUID or domain
- Converted to internal `ShopIdentifier` type for service layer

**Lookup Behavior**:
- **For UUID**: Direct lookup by shop ID (primary key)
- **For domain**: Lookup via domain index to find shop ID, then retrieve shop
- Both methods return the same complete shop data
- Domain lookup may return `SHOP_NOT_FOUND` if domain is not registered

**API Gateway Route Changes**:
- Route updated in CloudFormation: `GET /api/v1/shops/{shopIdentifier}`
- Route updated in CloudFormation: `PATCH /api/v1/shops/{shopIdentifier}`
- Path parameter name changed from `shopId` to `shopIdentifier` in API Gateway configuration

### Migration Guide

For frontend developers integrating these changes:

1. **Update API Endpoint URLs**:
   ```typescript
   // Before
   const getShopUrl = (shopId: string) => 
     `https://api.aura-historia.com/api/v1/shops/${shopId}`;
   
   // After
   const getShopUrl = (shopIdentifier: string) => 
     `https://api.aura-historia.com/api/v1/shops/${shopIdentifier}`;
   ```

2. **Flexible Shop Identification**:
   ```typescript
   // Can now use either UUID or domain
   await getShop("550e8400-e29b-41d4-a716-446655440000"); // UUID
   await getShop("tech-store-premium.com");              // Domain
   ```

3. **Update Error Handling**:
   ```typescript
   try {
     const shop = await getShop(identifier);
   } catch (error) {
     if (error.code === 'INVALID_SHOP_IDENTIFIER') {
       // Handle invalid shop identifier format
       showError('Invalid shop identifier. Please provide a valid UUID or domain.');
     } else if (error.code === 'SHOP_NOT_FOUND') {
       // Shop doesn't exist or domain not registered
       showError('Shop not found.');
     }
   }
   ```

4. **Parameter Validation**:
   ```typescript
   const isValidShopIdentifier = (identifier: string): boolean => {
     // Check if it's a UUID
     const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
     if (uuidRegex.test(identifier)) return true;
     
     // Check if it's a valid domain (contains at least one dot)
     const domainRegex = /^[a-z0-9]+([\-\.]{1}[a-z0-9]+)*\.[a-z]{2,}$/i;
     return domainRegex.test(identifier);
   };
   ```

5. **Update API Client Types**:
   ```typescript
   // Before
   interface GetShopParams {
     shopId: string; // UUID
   }
   
   // After
   interface GetShopParams {
     shopIdentifier: string; // UUID or domain
   }
   ```

### Benefits

**For Frontend Developers**:
- More intuitive shop references using human-readable domains
- Flexibility to use whichever identifier is available
- Simplifies deep linking and URL construction
- No need to store/retrieve shop UUID when domain is known

**For API Consumers**:
- Single endpoint supports multiple identification methods
- Backward compatible (UUIDs still work)
- Reduces need for separate domain-to-ID lookup calls
- More RESTful and user-friendly API design

**Use Cases**:
- **UUID**: Internal system references, database relations, unique identification
- **Domain**: User-facing features, shop discovery, URL construction, external integrations

### Removed

No endpoints, parameters, or functionality have been removed. This is a backward-compatible enhancement that adds domain-based identification while preserving UUID-based identification.

The error code `INVALID_UUID` is no longer returned for shop identifier validation failures. It has been replaced by the more general `INVALID_SHOP_IDENTIFIER` error code.

## 2025-12-10 - User Account Management

This update introduces comprehensive user account management capabilities, allowing authenticated users to view and update their account information including personal details, language preferences, and currency preferences.

### Added

#### GET /api/v1/me/account

Retrieves the authenticated user's account information.

**Endpoint Details**:
- **Method**: GET
- **Path**: `/api/v1/me/account`
- **Authentication**: Required (Cognito JWT)

**Response: 200 OK**

Returns complete user account data.

**Response Headers**:
- `Last-Modified` (string, HTTP date): When the user account was last updated
  - Example: `Wed, 01 Jan 2024 12:00:00 GMT`
- `Access-Control-Allow-Origin` (string): CORS header
  - Example: `*`

**Response Body**: `GetUserAccountData`

**Example Response**:
```json
{
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "firstName": "John",
  "lastName": "Doe",
  "language": "en",
  "currency": "EUR",
  "created": "2024-01-01T10:00:00Z",
  "updated": "2024-01-01T12:00:00Z"
}
```

**Error Responses**:

**401 Unauthorized**:
- `UNAUTHORIZED`: Missing or invalid JWT token

**404 Not Found**:
- `USER_NOT_FOUND`: User account does not exist

**500 Internal Server Error**:
- `INTERNAL_SERVER_ERROR`: Unexpected server error

#### PATCH /api/v1/me/account

Updates the authenticated user's account information.

**Endpoint Details**:
- **Method**: PATCH
- **Path**: `/api/v1/me/account`
- **Authentication**: Required (Cognito JWT)
- **Partial Updates**: All fields in request body are optional - only provided fields will be updated

**Request Body**: `PatchUserAccountData` (required, but all fields optional)
- `email` (string, email, optional): New email address
- `firstName` (string, optional, max 64 characters): New first name
- `lastName` (string, optional, max 64 characters): New last name
- `language` (LanguageData, optional): New preferred language
- `currency` (CurrencyData, optional): New preferred currency

**Example Requests**:

Update all fields:
```json
{
  "email": "newemail@example.com",
  "firstName": "Jane",
  "lastName": "Smith",
  "language": "de",
  "currency": "USD"
}
```

Update name only:
```json
{
  "firstName": "Jane",
  "lastName": "Smith"
}
```

Update preferences only:
```json
{
  "language": "fr",
  "currency": "GBP"
}
```

**Response: 200 OK**

Returns the updated user account data.

**Response Headers**:
- `Last-Modified` (string, HTTP date): When the user account was last updated
  - Example: `Wed, 01 Jan 2024 12:30:00 GMT`
- `Access-Control-Allow-Origin` (string): CORS header
  - Example: `*`

**Response Body**: `GetUserAccountData`

**Example Response**:
```json
{
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "email": "newemail@example.com",
  "firstName": "Jane",
  "lastName": "Smith",
  "language": "de",
  "currency": "USD",
  "created": "2024-01-01T10:00:00Z",
  "updated": "2024-01-01T12:30:00Z"
}
```

**Error Responses**:

**400 Bad Request**:
- `BAD_BODY_VALUE`: Missing or invalid request body
  - Examples: Empty body, malformed JSON

**401 Unauthorized**:
- `UNAUTHORIZED`: Missing or invalid JWT token

**404 Not Found**:
- `USER_NOT_FOUND`: User account does not exist

**500 Internal Server Error**:
- `INTERNAL_SERVER_ERROR`: Unexpected server error

#### New Data Types

**GetUserAccountData**
- Complete user account information
- Properties:
  - `userId` (string, UUID, required): Unique identifier for the user
  - `email` (string, email, required): User's email address
  - `firstName` (string, optional, max 64 characters): User's first name
  - `lastName` (string, optional, max 64 characters): User's last name
  - `language` (LanguageData, optional): User's preferred language (de, en, fr, es)
  - `currency` (CurrencyData, optional): User's preferred currency (EUR, GBP, USD, AUD, CAD, NZD)
  - `created` (string, date-time, required): When the user account was created (RFC3339)
  - `updated` (string, date-time, required): When the user account was last updated (RFC3339)

**PatchUserAccountData**
- Request body schema for updating user account
- All fields are optional - partial updates supported
- Properties:
  - `email` (string, email, optional): New email address
  - `firstName` (string, optional, max 64 characters): New first name
  - `lastName` (string, optional, max 64 characters): New last name
  - `language` (LanguageData, optional): New preferred language
  - `currency` (CurrencyData, optional): New preferred currency

#### New Error Codes

**USER_EXISTS_ALREADY** (409 Conflict)
- Returned when attempting to create a user that already exists
- Indicates the user ID is already in use in the system

**USER_NOT_FOUND** (404 Not Found)
- Returned when attempting to retrieve or update a user account that does not exist
- Applies to both GET and PATCH operations on `/api/v1/me/account`

### Implementation Notes

**User Account Structure**:
- User accounts are stored in DynamoDB with partition key format: `user#{userId}`
- The `userId` is derived from Cognito's JWT token `sub` claim
- All fields except `userId`, `email`, `created`, and `updated` are optional
- Field truncation: `firstName` and `lastName` are automatically truncated to 64 characters if longer
- Default values: When a user is created via Cognito sign-up, optional fields start as null

**Update Behavior**:
- PATCH operations support partial updates - only provided fields are modified
- Empty request body returns 400 error
- The `userId`, `created`, and `email` (when not provided in update) remain unchanged
- `updated` timestamp is set to current time on any modification
- Updates are performed using DynamoDB UpdateItem for atomicity

**Authentication**:
- Both endpoints require Cognito JWT authentication via `Authorization: Bearer <token>` header
- The `userId` is extracted from the JWT token's `sub` claim
- Invalid or missing tokens return 401 Unauthorized

**Field Constraints**:
- Email: Must be valid email format
- First Name: Optional, max 64 characters, auto-truncated if longer
- Last Name: Optional, max 64 characters, auto-truncated if longer
- Language: Must be one of: de, en, fr, es
- Currency: Must be one of: EUR, GBP, USD, AUD, CAD, NZD

### Migration Guide

For frontend developers integrating these changes:

1. **Retrieve User Account**:
   ```typescript
   const getUserAccount = async (accessToken: string): Promise<GetUserAccountData> => {
     const response = await fetch('https://api.aura-historia.com/api/v1/me/account', {
       method: 'GET',
       headers: {
         'Authorization': `Bearer ${accessToken}`,
       },
     });
     
     if (!response.ok) {
       throw new Error(`Failed to get user account: ${response.status}`);
     }
     
     return response.json();
   };
   ```

2. **Update User Account**:
   ```typescript
   const updateUserAccount = async (
     accessToken: string,
     updates: PatchUserAccountData
   ): Promise<GetUserAccountData> => {
     const response = await fetch('https://api.aura-historia.com/api/v1/me/account', {
       method: 'PATCH',
       headers: {
         'Authorization': `Bearer ${accessToken}`,
         'Content-Type': 'application/json',
       },
       body: JSON.stringify(updates),
     });
     
     if (!response.ok) {
       throw new Error(`Failed to update user account: ${response.status}`);
     }
     
     return response.json();
   };
   ```

3. **Handle Optional Fields**:
   ```typescript
   // Check if optional fields are present
   const displayName = (user: GetUserAccountData): string => {
     if (user.firstName && user.lastName) {
       return `${user.firstName} ${user.lastName}`;
     }
     if (user.firstName) {
       return user.firstName;
     }
     if (user.lastName) {
       return user.lastName;
     }
     return user.email;
   };
   ```

4. **Error Handling**:
   - Handle 401 errors by redirecting to login
   - Handle 404 errors by creating/initializing user profile
   - Handle 400 errors by validating input before sending

### Removed

No endpoints or schemas were removed in this update.

## 2025-12-08 - Shop Domain Normalization

This update changes how shop identifiers are handled in the API. Instead of storing and returning full URLs for shops, the system now uses normalized domains. This improves consistency, reduces storage overhead, and simplifies shop matching logic when enriching product data.

### Changed

#### Shop Data Structure - URLs Replaced with Domains

All shop-related data types now use `domains` instead of `urls`. Domains are normalized strings (lowercase, no scheme, no www prefix, no path/query/fragment) extracted from URLs.

**Affected Data Types**:
- `GetShopData`
- `PostShopData`
- `PatchShopData`

**Migration Guide**:

**Before (using URLs)**:
```json
{
  "shopId": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Tech Store Premium",
  "urls": [
    "https://tech-store-premium.com",
    "https://tech-store-premium.de",
    "https://apple.tech-store-premium.com"
  ]
}
```

**After (using domains)**:
```json
{
  "shopId": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Tech Store Premium",
  "domains": [
    "tech-store-premium.com",
    "tech-store-premium.de",
    "apple.tech-store-premium.com"
  ]
}
```

**Key Differences**:
- Field name changed from `urls` to `domains`
- Values are now plain domain strings, not full URLs
- Domains are normalized: lowercase, no scheme (`https://`), no `www.` prefix
- Paths, query parameters, and fragments are stripped
- Port numbers are stripped (e.g., `:8080`)

**Domain Normalization Examples**:
- `https://Tech-Store.COM/products` → `tech-store.com`
- `http://www.example.com/?ref=123` → `example.com`
- `https://shop.example.com/path#anchor` → `shop.example.com`

**Frontend Impact**:
- When creating/updating shops: You can send either full URLs or domain strings - both will be normalized
- When receiving shop data: Expect domain strings, not URLs
- For display: Add `https://` prefix if you need to show or link to the shop
- For comparison: Use exact string matching on normalized domains

**API Endpoints Affected**:
- `POST /api/v1/shops` - Request and response
- `PATCH /api/v1/shops/{shopId}` - Request and response
- `GET /api/v1/shops/{shopId}` - Response
- All shop search endpoints - Response

#### Error Code Changes

**Renamed Error Code**: `SHOP_TOO_MANY_URLS` → `SHOP_TOO_MANY_DOMAINS`
- **Endpoints**: `POST /api/v1/shops`, `PATCH /api/v1/shops/{shopId}`
- **HTTP Status**: 400 Bad Request
- **When**: More than 100 domains provided in request
- **Example Detail**: "Shop can only have 100 domains but was given more: '142'"

**Updated Error Messages**:
- `SHOP_EXISTS_ALREADY` detail message updated to reference domains instead of URLs
  - Old: "...an URL for any domain of shop is already registered"
  - New: "...a domain of shop is already registered"

### Added

#### New Error Code for Product Enrichment

**Error Code**: `NO_DOMAIN`
- **Endpoint**: `PUT /api/v1/products`
- **Type**: Product enrichment failure
- **When**: Product URL does not contain a valid extractable domain
- **Description**: The product URL could not be parsed to extract a domain for shop matching
- **Examples of URLs that fail**:
  - `https://localhost` (no TLD, just hostname)
  - `https://127.0.0.1` (IP address without domain)
  - `http://localhost:8080/product` (localhost with port)

**Response Example**:
```json
{
  "failed": {
    "https://localhost:8080/item": "NO_DOMAIN"
  },
  "skipped": 0
}
```

**Frontend Impact**:
- Handle `NO_DOMAIN` as a new error type in product upload responses
- These products cannot be processed and will need valid domain-based URLs
- Display appropriate error message to users

#### Enhanced Product Enrichment Documentation

The `PUT /api/v1/products` endpoint description now explicitly documents:
- Domain extraction from product URLs
- Shop matching based on extracted domains
- Two possible failures: `SHOP_NOT_FOUND` (domain not registered) and `NO_DOMAIN` (invalid URL)

### Technical Details

**Domain Extraction Logic**:
1. Remove `http://` or `https://` scheme
2. Remove `www.` prefix
3. Extract hostname (everything before `:`, `/`, `?`, or `#`)
4. Convert to lowercase
5. Validate: must contain at least one `.` (TLD required)

**Backend Type**: New `Domain` type in Rust
- Serializes/deserializes as a plain string (not URL)
- Implements validation and normalization on creation
- Used in shop records and shop-product matching logic

**Database Changes**:
- DynamoDB partition keys updated from `shop#url#{url}` to `shop#domain#{domain}`
- Shop records now indexed by normalized domain instead of full URL
- Product enrichment queries shops by domain extracted from product URL

## 2025-12-06 - Shop Management Endpoints

This update introduces comprehensive shop management capabilities, allowing creation and modification of shop entities in the system. Previously, shops could only be retrieved or searched - now they can be fully managed through the API.

### Added

#### POST /api/v1/shops

Creates a new shop in the system with the provided details.

**Endpoint Details**:
- **Method**: POST
- **Path**: `/api/v1/shops`
- **Authentication**: None (public endpoint)

**Request Body**: `PostShopData` (required)
- `name` (string, required, max 255 characters): Display name of the shop
- `domains` (array of strings, required, min 1, max 100): All domains associated with the shop
  - Can be provided as full URLs (will be normalized) or as domain strings
  - Domains are normalized to lowercase without scheme, www prefix, or path/query/fragment
  - At least one domain is required
  - Maximum 100 domains allowed per shop
- `image` (string, URI, optional): URL to the shop's logo or image

**Example Request**:
```json
{
  "name": "Tech Store Premium",
  "domains": [
    "tech-store-premium.com",
    "tech-store-premium.de",
    "apple.tech-store-premium.com"
  ],
  "image": "https://tech-store-premium.com/logo.svg"
}
```

**Response: 201 Created**

Returns the created shop with generated ID and timestamps.

**Response Headers**:
- `Location` (string, URI): URL of the created shop resource
  - Example: `https://api.aura-historia.com/api/v1/shops/550e8400-e29b-41d4-a716-446655440000`
- `Last-Modified` (string, HTTP date): When the shop was created
  - Example: `Wed, 01 Jan 2024 12:00:00 GMT`

**Response Body**: `GetShopData`

**Example Response**:
```json
{
  "shopId": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Tech Store Premium",
  "domains": [
    "tech-store-premium.com",
    "tech-store-premium.de",
    "apple.tech-store-premium.com"
  ],
  "image": "https://tech-store-premium.com/logo.svg",
  "created": "2024-01-01T12:00:00Z",
  "updated": "2024-01-01T12:00:00Z"
}
```

**Error Responses**:

**400 Bad Request**:
- `BAD_BODY_VALUE`: Missing or invalid request body
  - Examples: Empty body, malformed JSON, missing required fields
- `SHOP_TOO_MANY_DOMAINS`: More than 100 domains provided
  - Example detail: "Shop can only have 100 domains but was given more: '142'"

**409 Conflict**:
- `SHOP_EXISTS_ALREADY`: A shop with one of the provided domains already exists
  - Example detail: "Shop with name 'Tech Store Premium' exists already - a domain of shop is already registered"

**500 Internal Server Error**:
- `INTERNAL_SERVER_ERROR`: Unexpected server error

**503 Service Unavailable**:
- `UNPROCESSED_ITEMS`: Temporary database issue prevented completion
  - Example detail: "Did not succeed checking existence of shop due to DynamoDB Batch-Response containing unprocessed items"

#### PATCH /api/v1/shops/{shopId}

Updates an existing shop's information by its unique identifier.

**Endpoint Details**:
- **Method**: PATCH
- **Path**: `/api/v1/shops/{shopId}`
- **Authentication**: None (public endpoint)
- **Partial Updates**: All fields in request body are optional - only provided fields will be updated
- **No-op Behavior**: If request body is empty or only contains null values, shop is returned unchanged

**Path Parameters**:
- `shopId` (string, UUID, required): Unique identifier of the shop to update

**Request Body**: `PatchShopData` (required, but all fields optional)
- `name` (string, optional, max 255 characters): New display name for the shop
- `domains` (array of strings, optional, min 1, max 100): Complete new set of domains
  - Can be provided as full URLs (will be normalized) or as domain strings
  - **Important**: When updating domains, the complete new set must be provided (not a diff)
  - Old domains not in the new set will be removed
  - New domains in the set will be added
- `image` (string, URI, optional): New URL to the shop's logo or image

**Example Requests**:

Update only the name:
```json
{
  "name": "Tech Store Premium Plus"
}
```

Update domains (complete replacement):
```json
{
  "domains": [
    "tech-store-premium.com",
    "tech-store-premium.eu"
  ]
}
```

Update multiple fields:
```json
{
  "name": "Tech Store Premium Plus",
  "domains": [
    "tech-store-premium.com",
    "tech-store-premium.eu"
  ],
  "image": "https://tech-store-premium.com/new-logo.svg"
}
```

**Response: 200 OK**

Returns the updated shop with modified fields and new `updated` timestamp.

**Response Headers**:
- `Last-Modified` (string, HTTP date): When the shop was last updated
  - Example: `Wed, 01 Jan 2024 12:30:00 GMT`

**Response Body**: `GetShopData`

**Example Response**:
```json
{
  "shopId": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Tech Store Premium Plus",
  "domains": [
    "tech-store-premium.com",
    "tech-store-premium.eu"
  ],
  "image": "https://tech-store-premium.com/new-logo.svg",
  "created": "2024-01-01T10:00:00Z",
  "updated": "2024-01-01T12:30:00Z"
}
```

**Error Responses**:

**400 Bad Request**:
- `BAD_PATH_PARAMETER_VALUE`: Missing shop ID in path
- `INVALID_UUID`: Invalid UUID format for shop ID
- `BAD_BODY_VALUE`: Missing or invalid request body
- `SHOP_TOO_MANY_DOMAINS`: More than 100 domains provided

**404 Not Found**:
- `SHOP_NOT_FOUND`: Shop with the given ID does not exist

**500 Internal Server Error**:
- `INTERNAL_SERVER_ERROR`: Unexpected server error

**503 Service Unavailable**:
- `UNPROCESSED_ITEMS`: Temporary database issue prevented completion

#### New Data Types

**PostShopData**
- Request body schema for creating a new shop
- All fields that will be stored for the shop except generated fields (shopId, created, updated)
- Properties:
  - `name` (string, required, max 255 characters): Shop display name
  - `domains` (array of strings, required, min 1, max 100): All shop domains (normalized)
  - `image` (string, URI, optional): Shop logo or image URL

**PatchShopData**
- Request body schema for updating an existing shop
- All fields are optional - partial updates supported
- Properties:
  - `name` (string, optional, max 255 characters): New shop name
  - `domains` (array of strings, optional, min 1, max 100): Complete new set of domains (normalized)
  - `image` (string, URI, optional): New shop logo or image URL

#### New Error Codes

**SHOP_EXISTS_ALREADY** (409 Conflict)
- Returned when attempting to create a shop with a domain that is already registered to another shop
- Ensures domain uniqueness across all shops in the system
- Error message includes the shop name that was attempted to be created
- Example: "Shop with name 'Tech Store Premium' exists already - a domain of shop is already registered"

**SHOP_TOO_MANY_DOMAINS** (400 Bad Request)
- Returned when request contains more than 100 domains
- Applies to both POST (create) and PATCH (update) operations
- Error message includes the actual number of domains provided
- Example: "Shop can only have 100 domains but was given more: '142'"

**UNPROCESSED_ITEMS** (503 Service Unavailable)
- Returned when database batch operations cannot complete due to unprocessed items
- Temporary error that may resolve on retry
- Indicates the system is temporarily unable to process the request due to database limitations
- Example detail: "Did not succeed checking existence of shop due to DynamoDB Batch-Response containing unprocessed items"

### Changed

#### GetShopData Schema

The `urls` field has been enhanced with validation constraints:

**Before**:
```yaml
urls:
  type: array
  items:
    type: string
    format: uri
  description: All known URLs to the shop's website
  default: []
```

**After**:
```yaml
urls:
  type: array
  items:
    type: string
    format: uri
  description: All known URLs to the shop's website
  minItems: 1
  maxItems: 100
```

**Changes**:
- **Removed** `default: []` - shops must always have at least one URL
- **Added** `minItems: 1` - enforces that shops have at least one URL
- **Added** `maxItems: 100` - enforces maximum of 100 URLs per shop

**Migration Impact**:
- No impact on existing API consumers - this is a documentation clarification of existing backend constraints
- Frontend validation should ensure at least 1 URL and maximum 100 URLs when creating or updating shops

The `name` field also received a constraint:
- **Added** `maxLength: 255` - shop names are limited to 255 characters

**Migration Impact**:
- No impact on existing API consumers - this is a documentation clarification
- Frontend validation should limit shop name input to 255 characters

### Implementation Notes

**Shop URL Management**:
- Each shop URL creates a separate lookup record in DynamoDB
- URLs are used for automatic shop identification during product ingestion
- When updating URLs via PATCH, old URL records are deleted and new ones are created
- All URL records for a shop maintain consistency with the shop's core data

**Uniqueness Constraints**:
- Shop URLs must be globally unique across all shops
- A URL can only be associated with one shop at a time
- Creating or updating a shop with a URL that exists for another shop returns `SHOP_EXISTS_ALREADY` error

**Update Behavior**:
- PATCH operations support partial updates - only provided fields are modified
- Empty request body or all-null fields result in no changes (200 OK with unchanged shop)
- URL updates replace the entire URL set (not a merge)
- `created` timestamp never changes
- `updated` timestamp is set to current time on any modification

**Performance Characteristics**:
- Shop creation performs existence check for all provided URLs before creating
- Up to 101 DynamoDB operations (1 shop record + up to 100 URL records)
- All operations are transactional - either all succeed or all fail
- Update operations may delete old URL records and create new ones

### Removed

No endpoints or schemas were removed in this update.

## 2025-11-23 - Item-Product-Renaming & ApiError problem+json

### Changed

- Renamed all item-related types to product-version
- Adjusted all error-responses with response-payload `ApiError` to return `application/problem+json` (RFC 9457) instead of `application/json`
  - Renamed field `message` to `detail` (human-readable for devs - not user-facing)
  - Added field `title` (human-readable for devs - not user-facing)

## 2025-11-10 - Similar Items Endpoint with Semantic Search

This update introduces a new endpoint for finding semantically similar items using k-nearest neighbors (k-NN) search on text embeddings. The endpoint leverages machine learning embeddings to provide relevant item recommendations based on content similarity.

### Added

#### GET /api/v1/items/{shopId}/{shopsItemId}/similar

Retrieves items similar to the specified item using semantic search based on text embeddings.

**Endpoint Details**:
- **Method**: GET
- **Path**: `/api/v1/items/{shopId}/{shopsItemId}/similar`
- **Authentication**: Optional (Cognito JWT Bearer token)
- **Supported Languages**: de, en, fr, es (via Accept-Language header)
- **Supported Currencies**: EUR, USD, GBP, AUD, CAD, NZD (via currency query parameter)

**How It Works**:
- Uses k-nearest neighbors (k-NN) search on text embeddings to find semantically similar items
- Text embeddings are 1024-dimensional vectors computed from item titles and descriptions using the baai/bge-m3 model
- Embeddings are generated nightly via batch processing for all items
- Returns up to 20 similar items, ranked by similarity score
- The original item is automatically excluded from results

**Path Parameters**:
- `shopId` (string, uuid, required): Unique identifier of the shop
- `shopsItemId` (string, required): Shop's unique identifier for the item

**Query Parameters**:
- `currency` (CurrencyData, optional): Currency for price display (default: EUR)
  - Accepted values: `EUR`, `USD`, `GBP`, `AUD`, `CAD`, `NZD`

**Request Headers**:
- `Accept-Language` (optional): Preferred language for localized content
  - Format: Language code with optional quality values
  - Examples: `de`, `en;q=0.9,de;q=0.8`, `en-US`
- `Authorization` (optional): Cognito JWT token for personalized response
  - Format: `Bearer [JWT token]`
  - When provided: Response includes user-specific state for each similar item
  - When omitted: Response contains only item data without user state

**Response: 200 OK**

Returns an array of similar items, each following the PersonalizedGetItemData schema.

**Schema**: Array of `PersonalizedGetItemData`

Each item in the array contains:
- `item` (GetItemData, required): Complete item information
  - `itemId` (string, uuid, required): Unique item identifier
  - `eventId` (string, uuid, required): Latest event identifier
  - `shopId` (string, uuid, required): Shop identifier
  - `shopsItemId` (string, required): Shop's item identifier
  - `shopName` (string, required): Shop name
  - `title` (LocalizedTextData, required): Localized title
  - `description` (LocalizedTextData, optional): Localized description
  - `price` (PriceData, optional): Price in requested currency
  - `state` (ItemStateData, required): Item state
  - `url` (string, uri, required): Item URL
  - `images` (array of strings, required): Image URLs
  - `created` (string, date-time, required): Creation timestamp
  - `updated` (string, date-time, required): Last update timestamp
- `userState` (ItemUserStateData, optional): User-specific state (only when authenticated)
  - `watchlist` (WatchlistUserStateData, required): Watchlist state
    - `watching` (boolean, required): Whether item is on user's watchlist
    - `notifications` (boolean, required): Whether notifications are enabled

**Example Response (Authenticated User)**:
```json
[
  {
    "item": {
      "itemId": "660e8400-e29b-41d4-a716-446655440001",
      "eventId": "660e8400-e29b-41d4-a716-446655440011",
      "shopId": "550e8400-e29b-41d4-a716-446655440000",
      "shopsItemId": "6ba7b811",
      "shopName": "My Shop",
      "title": {
        "text": "Similar Product",
        "language": "en"
      },
      "description": {
        "text": "This product has similar features",
        "language": "en"
      },
      "price": {
        "currency": "EUR",
        "amount": 2899
      },
      "state": "AVAILABLE",
      "url": "https://my-shop.com/products/similar-product",
      "images": [
        "https://my-shop.com/images/similar-1.jpg"
      ],
      "created": "2024-01-01T10:00:00Z",
      "updated": "2024-01-01T12:00:00Z"
    },
    "userState": {
      "watchlist": {
        "watching": true,
        "notifications": false
      }
    }
  },
  {
    "item": {
      "itemId": "770e8400-e29b-41d4-a716-446655440002",
      "eventId": "770e8400-e29b-41d4-a716-446655440012",
      "shopId": "550e8400-e29b-41d4-a716-446655440000",
      "shopsItemId": "6ba7b812",
      "shopName": "My Shop",
      "title": {
        "text": "Another Similar Item",
        "language": "en"
      },
      "price": {
        "currency": "EUR",
        "amount": 3199
      },
      "state": "AVAILABLE",
      "url": "https://my-shop.com/products/another-item",
      "images": [],
      "created": "2024-01-01T11:00:00Z",
      "updated": "2024-01-01T13:00:00Z"
    },
    "userState": {
      "watchlist": {
        "watching": false,
        "notifications": false
      }
    }
  }
]
```

**Example Response (Anonymous User)**:
```json
[
  {
    "item": {
      "itemId": "660e8400-e29b-41d4-a716-446655440001",
      "eventId": "660e8400-e29b-41d4-a716-446655440011",
      "shopId": "550e8400-e29b-41d4-a716-446655440000",
      "shopsItemId": "6ba7b811",
      "shopName": "My Shop",
      "title": {
        "text": "Similar Product",
        "language": "en"
      },
      "price": {
        "currency": "EUR",
        "amount": 2899
      },
      "state": "AVAILABLE",
      "url": "https://my-shop.com/products/similar-product",
      "images": [
        "https://my-shop.com/images/similar-1.jpg"
      ],
      "created": "2024-01-01T10:00:00Z",
      "updated": "2024-01-01T12:00:00Z"
    }
  }
]
```

**Response: 202 Accepted**

Returned when the item's text embedding has not yet been computed (typically for items less than 24 hours old).
Embeddings are generated during nightly batch processing.

**Headers**:
- `Location` (string, uri): URL to poll for similar items once embeddings are computed
  - Example: `https://api.aura-historia.com/api/v1/items/550e8400-e29b-41d4-a716-446655440000/6ba7b810/similar`

**Response Body**: None

**When This Occurs**:
- Item was created less than 24 hours ago and hasn't been processed by nightly embedding generation
- Item's text embedding is missing from the OpenSearch index
- Embedding generation is scheduled but not yet completed

**Migration Impact**:
- Frontend should implement polling logic when receiving 202 response
- Use exponential backoff to avoid excessive requests
- Check again after 24 hours or on next user visit
- Display appropriate message to users (e.g., "Similar items will be available soon")

**Response: 400 Bad Request**

Returned when request parameters are invalid.

**Error Codes**:
- `BAD_PATH_PARAMETER_VALUE`: Missing or invalid path parameter
- `INVALID_UUID`: Invalid UUID format for shopId
- `BAD_QUERY_PARAMETER_VALUE`: Invalid query parameter value
- `INVALID_CURRENCY`: Invalid currency code

**Example**:
```json
{
  "status": 400,
  "error": "BAD_PATH_PARAMETER_VALUE",
  "source": {
    "field": "shopId",
    "sourceType": "path"
  }
}
```

**Response: 404 Not Found**

Returned when the specified item does not exist.

**Error Code**: `ITEM_NOT_FOUND`

**Example**:
```json
{
  "status": 404,
  "error": "ITEM_NOT_FOUND",
  "message": "Item with ShopId '550e8400-e29b-41d4-a716-446655440000' and ShopsItemId '6ba7b810' not found."
}
```

**Response: 500 Internal Server Error**

Returned when an unexpected server error occurs.

**Error Code**: `INTERNAL_SERVER_ERROR`

**Technical Details**:

The similar items feature is powered by:
- **Embedding Model**: baai/bge-m3 (1024-dimensional vectors)
- **Search Method**: k-NN (k-nearest neighbors) using cosine similarity
- **Index**: OpenSearch with KNN plugin
- **Field Name**: `textEmbedding` in the items index
- **Batch Processing**: Nightly enrichment pipeline generates embeddings for items without them
- **Maximum Results**: 20 similar items per request
- **Exclusion**: Original item is automatically filtered from results

**Use Cases**:
- Product recommendations on item detail pages
- "You might also like" sections
- Discovery of related items across different shops
- Content-based filtering for personalization

**Performance Characteristics**:
- Fast k-NN search using approximate nearest neighbors (ANN)
- Sub-second response times for most queries
- Scalable to millions of items
- Real-time personalization when authenticated

**Limitations**:
- Requires items to have been processed by nightly embedding generation (24-hour delay for new items)
- Similarity is based purely on text content (title + description)
- Does not consider user behavior, purchase history, or collaborative filtering
- Limited to 20 results per request
- Results are not paginated (returns all similar items up to limit)

### Changed

No existing endpoints or schemas were modified in this update.

### Removed

No endpoints or schemas were removed in this update.

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

#### Renamed Search-Filters

- A `searchFilter` no longer exists. The object used for searching items is now called `ItemSearchData` along with its field `itemSearch`
- User-stored search-filters are now called `UserSearchFilterData`. 

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
- `GET /api/v1/search-filters/{userSearchFilterId}` → `GET /api/v1/me/search-filters/{userSearchFilterId}`
- `PATCH /api/v1/search-filters/{userSearchFilterId}` → `PATCH /api/v1/me/search-filters/{userSearchFilterId}`
- `DELETE /api/v1/search-filters/{userSearchFilterId}` → `DELETE /api/v1/me/search-filters/{userSearchFilterId}`

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

### Removed

#### GET /api/v1/items (Simple Text Search)

The simple text search endpoint has been removed. Use `POST /api/v1/items/search` instead for all search operations.

**Removed Endpoint**:
- `GET /api/v1/items?q={query}&language={lang}&currency={cur}&sort={field}&order={order}&searchAfter={cursor}&size={size}`

**Migration Path**:
Use the complex search endpoint (`POST /api/v1/items/search`) which provides all the functionality of the simple search and more.

**Before** (Simple Search):
```http
GET /api/v1/items?q=smartphone+case&language=en&currency=USD&sort=price&order=asc&size=21
```

**After** (Complex Search):
```http
POST /api/v1/items/search?sort=price&order=asc&size=21
Content-Type: application/json

{
  "language": "en",
  "currency": "USD",
  "itemQuery": "smartphone case"
}
```

**Migration Impact**:
- All clients using simple search must migrate to complex search
- Query parameters `q`, `language`, and `currency` move to request body
- Sort and pagination parameters remain as query parameters
- Response structure is identical (wrapped in `PersonalizedItemSearchResultData` with authentication)
- Additional filtering options are available through the request body (shopNameQuery, price range, state, date ranges)

**Example Request/Response**:

Before (Simple Search):
```http
GET /api/v1/items?q=smartphone+case&language=en&currency=USD&sort=price&order=asc&size=21
```

After (Complex Search):
```http
POST /api/v1/items/search?sort=price&order=asc&size=21
Content-Type: application/json

{
  "language": "en",
  "currency": "USD",
  "itemQuery": "smartphone case"
}
```

Response (identical structure, wrapped in PersonalizedItemSearchResultData):
```json
{
  "items": [
    {
      "item": { /* GetItemData */ },
      "userState": { /* Optional when authenticated */ }
    }
  ],
  "size": 21,
  "total": 127,
  "searchAfter": "[2999, \"550e8400-e29b-41d4-a716-446655440000\"]"
}
```

No other endpoints or schemas have been removed in this update. The old watchlist and search-filter paths at `/api/v1/watchlist*` and `/api/v1/search-filters*` are deprecated and will be removed in a future update. Clients should migrate to the new `/api/v1/me/*` paths immediately.

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
