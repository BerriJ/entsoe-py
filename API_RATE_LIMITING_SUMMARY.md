# API Rate Limitation Handling in entsoe-py

This document provides a comprehensive summary of how the entsoe-py package handles API rate limitations and ensures robust data retrieval from the ENTSO-E API.

## Overview

The entsoe-py package implements multiple layers of protection against API rate limits, connection issues, and data volume restrictions. These mechanisms work automatically and transparently, requiring no additional configuration from users in most cases.

## Rate Limiting Strategies

### 1. Connection Retry Mechanism (`@retry` decorator)

**Location**: `entsoe/decorators.py` (lines 15-41)  
**Applied to**: `_base_request()` method in `EntsoeRawClient`

- **Purpose**: Handles temporary connection failures and network issues
- **Configurable parameters**:
  - `retry_count`: Number of retry attempts (default: 3)
  - `retry_delay`: Wait time between retries in seconds (default: 10)
- **Handles exceptions**:
  - `requests.ConnectionError`
  - `socket.gaierror` (DNS resolution errors)
  - `http.client.RemoteDisconnected`
- **Behavior**: When a connection error occurs, the decorator logs a warning and waits for the specified delay before retrying

### 2. Pagination Handling (`@paginated` decorator)

**Location**: `entsoe/decorators.py` (lines 44-59)  
**Applied to**: Methods that may request large datasets

- **Purpose**: Automatically handles `PaginationError` when too much data is requested
- **Strategy**: Recursively splits the requested time period in half until each request succeeds
- **Process**:
  1. Attempts the original request
  2. If `PaginationError` occurs, calculates a pivot point (midpoint of time range)
  3. Makes two separate requests for each half of the time period
  4. Concatenates the results using `pd.concat()`

### 3. Time-based Limitations

#### Year Limited (`@year_limited` decorator)

**Location**: `entsoe/decorators.py` (lines 101-162)  
**Applied to**: Most query methods in `EntsoePandasClient`

- **Purpose**: Handles API restrictions on queries spanning more than one year
- **Strategy**: Splits requests into yearly blocks using `year_blocks()` function
- **Features**:
  - Validates that start and end parameters are timezone-aware pandas Timestamps
  - Handles partial data overlaps by truncating frames to avoid duplication
  - Uses left-open intervals except for the first block
  - Gracefully handles `NoMatchingDataError` for individual blocks

#### Day Limited (`@day_limited` decorator)

**Location**: `entsoe/decorators.py` (lines 165-190)  
**Applied to**: Specific methods requiring daily data chunks

- **Purpose**: Splits requests into daily blocks when needed
- **Strategy**: Uses `day_blocks()` function to create daily time ranges
- **Error handling**: Continues processing even if some daily blocks return no data

### 4. Document Count Limitations (`@documents_limited(n)` decorator)

**Location**: `entsoe/decorators.py` (lines 62-91)  
**Applied to**: Methods with `@documents_limited(100)` (common limit)

- **Purpose**: Handles API restrictions on the number of documents per request
- **Strategy**: Uses offset-based pagination to retrieve data in chunks
- **Process**:
  1. Iterates through offsets in steps of `n` (typically 100)
  2. Maximum offset limit: 4800 + n
  3. Stops when `NoMatchingDataError` occurs (indicating no more data)
  4. Concatenates all successful responses
  5. Handles duplicate indices by keeping the last valid value

### 5. API Error Detection and Handling

**Location**: `entsoe/entsoe.py` `_base_request()` method (lines 121-134)

The package parses API error messages to detect specific rate limit violations:

#### Detected Error Patterns:
- `"amount of requested data exceeds allowed limit"`
- `"requested data to be gathered via the offset parameter exceeds the allowed limit"`

#### Error Processing:
- Extracts allowed and requested quantities from error messages
- Raises `PaginationError` with descriptive messages
- Enables automatic handling by `@paginated` decorator

### 6. Token Management (File Client)

**Location**: `entsoe/files/decorators.py` (`@check_expired` decorator)  
**Applied to**: All `EntsoeFileClient` API methods

- **Purpose**: Prevents authentication failures due to expired tokens
- **Strategy**: Automatically refreshes access tokens before they expire
- **Implementation**: Checks token expiration before each API call and refreshes if needed

## Configuration Options

### Client Initialization Parameters

```python
client = EntsoeRawClient(
    api_key="your_api_key",
    retry_count=3,        # Number of retry attempts
    retry_delay=10,       # Seconds to wait between retries
    timeout=None,         # Request timeout
    proxies=None          # HTTP proxies
)
```

### Rate Limiting Parameters
- Most rate limiting is handled automatically with sensible defaults
- `@documents_limited(100)`: Document limit can be customized per endpoint
- Time-based limits are determined by the API's inherent restrictions

## Exception Types

**Location**: `entsoe/exceptions.py`

- `PaginationError`: Raised when request exceeds API data limits
- `NoMatchingDataError`: Raised when no data is available for the requested parameters
- Other domain-specific exceptions for invalid parameters

## Automatic Behavior

The package automatically:
1. **Retries failed connections** with exponential backoff
2. **Splits large time ranges** into manageable chunks
3. **Handles pagination** for document-heavy requests
4. **Manages authentication tokens** for file downloads
5. **Combines results** from multiple API calls into unified datasets
6. **Preserves data integrity** by handling overlaps and duplicates

## Best Practices for Users

1. **Use timezone-aware timestamps**: Required for time-limited decorators
2. **Specify reasonable time ranges**: While automatic splitting occurs, smaller ranges are more efficient
3. **Handle NoMatchingDataError**: This exception indicates legitimately missing data, not a rate limit issue
4. **Monitor logs**: The package logs warnings for connection retries and debug messages for data splitting

## Technical Implementation Details

- **Logging**: Uses Python's logging module with `entsoe` logger
- **Data handling**: Leverages pandas for efficient data concatenation and deduplication
- **Error parsing**: Uses BeautifulSoup to parse XML error responses
- **Time handling**: Utilizes pandas Timestamps with timezone support

This comprehensive rate limiting system ensures robust data retrieval while respecting API constraints and providing a seamless user experience.