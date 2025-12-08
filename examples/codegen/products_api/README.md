# Products API (codegen)

Comprehensive REST API example demonstrating KATANA codegen capabilities with a realistic in-memory product catalog.

## What This Demonstrates

### Codegen Features
- **Complex DTOs**: Nested objects with validation (Product, ProductList, CreateProductRequest, UpdateProductRequest)
- **Enum support**: ProductCategory enum → `enum class` code generation
- **Array handling**: Tags array with maxItems constraint
- **Nullable fields**: Optional description field
- **Path parameters**: RESTful ID-based routing
- **Query parameters**: Pagination (limit, offset) and search
- **Request validation**: Pattern matching (SKU), min/max constraints, required fields
- **Multiple HTTP methods**: GET, POST, PUT, DELETE on same path
- **Error handling**: 400, 404, 409 with Problem Details (RFC 7807)

### Architecture Patterns
- **In-memory storage**: Simple `std::unordered_map` without complex indexing
- **Arena allocators**: All request DTOs use per-request arena allocation
- **Zero-copy JSON**: Streaming parser/serializer with string_view where possible
- **Type safety**: Compile-time route binding and parameter extraction
- **Hot-path optimization**: No virtual dispatch in request handling

## Not Included (By Design)
- ❌ No SQL/NoSQL database
- ❌ No complex query engine or indexing
- ❌ No persistent storage
- ❌ No caching layer
- ❌ No authentication/authorization

The focus is **pure codegen demonstration**: DTO validation, JSON ser/des, routing, and error handling.

## Build and Run

```bash
cmake --preset examples
cmake --build --preset examples --target products_api
./build/examples/examples/codegen/products_api/products_api  # PORT=8082 by default
```

## API Examples

### Create Product
```bash
curl -X POST http://localhost:8082/products \
  -H 'Content-Type: application/json' \
  -d '{
    "sku": "LAPTOP-001",
    "name": "ThinkPad X1 Carbon",
    "description": "Lightweight business laptop",
    "price": 1299.99,
    "stock": 50,
    "category": "electronics",
    "tags": ["laptop", "business", "portable"]
  }'
# => {"id":1,"sku":"LAPTOP-001","name":"ThinkPad X1 Carbon",...}
```

### List Products (Paginated)
```bash
curl http://localhost:8082/products?limit=5&offset=0
# => {"items":[...],"total":10,"limit":5,"offset":0}
```

### Get Product by ID
```bash
curl http://localhost:8082/products/1
# => {"id":1,"sku":"LAPTOP-001",...}
```

### Update Product
```bash
curl -X PUT http://localhost:8082/products/1 \
  -H 'Content-Type: application/json' \
  -d '{"price": 1199.99, "stock": 45}'
# => {"id":1,...,"price":1199.99,"stock":45}
```

### Search Products
```bash
curl 'http://localhost:8082/products/search?query=laptop'
# => [{"id":1,"name":"ThinkPad X1 Carbon",...}]
```

### Adjust Stock
```bash
curl -X POST http://localhost:8082/products/1/stock \
  -H 'Content-Type: application/json' \
  -d '{"delta": -5}'
# => {"new_stock":40}
```

### Delete Product
```bash
curl -X DELETE http://localhost:8082/products/1
# => 204 No Content
```

## Validation Examples

Invalid SKU pattern:
```bash
curl -X POST http://localhost:8082/products \
  -H 'Content-Type: application/json' \
  -d '{"sku":"invalid","name":"Test","price":10,"category":"other"}'
# => 400 Bad Request with Problem Details
```

Duplicate SKU:
```bash
curl -X POST http://localhost:8082/products \
  -H 'Content-Type: application/json' \
  -d '{"sku":"LAPTOP-001","name":"Duplicate","price":10,"category":"other"}'
# => 409 Conflict
```

Product not found:
```bash
curl http://localhost:8082/products/999
# => 404 Not Found with Problem Details
```

## Benchmark Integration

This service is designed to be included in the benchmark suite:

### Latency Benchmarks
- Measure p50/p95/p99/p999 for each endpoint
- Vary payload sizes (minimal vs full DTO with tags/description)
- Test with different stock levels (0, 100, 1000000)

### Throughput Benchmarks
- Single-threaded: pure handler performance
- Multi-threaded (4/8/16 threads): scalability
- Mixed workload: 70% reads (GET), 20% updates (PUT), 10% creates (POST)

### Validation Benchmarks
- Valid requests (should be fast path)
- Invalid requests (pattern validation, range checks)
- Missing required fields
- Success rate tracking

### Recommended Test Scenarios

**Scenario 1: Pure GET (hot cache)**
- Pre-populate 100 products
- Random GET /products/{id} from 1-100
- Expected: p99 < 0.15 ms, 300k+ rps

**Scenario 2: Mixed CRUD**
- 60% GET, 20% POST, 15% PUT, 5% DELETE
- Expected: p99 < 0.25 ms, 150k+ rps

**Scenario 3: Validation stress**
- 50% valid, 50% invalid requests
- Mix of validation failure types
- Expected: p99 < 0.20 ms (validation is fast)

**Scenario 4: Search workload**
- GET /products/search with various query strings
- Expected: p99 < 0.30 ms (linear scan in-memory)

These scenarios test:
- Arena allocation overhead
- JSON parse/serialize performance
- Router dispatch efficiency
- Validation constraint checking
- Error path performance (400/404/409)

## Memory Profile

Per request (typical):
- Arena allocation: ~2-4 KB for average Product DTO
- Peak arena usage: ~8 KB for ProductList with 10 items
- Zero heap allocations in hot path

## Performance Expectations (x86-64, single reactor thread)

- **GET /products/{id}**: p99 < 0.15 ms, 350k rps
- **POST /products**: p99 < 0.25 ms, 180k rps (with validation)
- **PUT /products/{id}**: p99 < 0.20 ms, 200k rps
- **GET /products?limit=10**: p99 < 0.30 ms, 150k rps
- **GET /products/search**: p99 < 0.35 ms, 120k rps

Multi-threaded (8 reactors):
- **Aggregate throughput**: 800k-1.2M rps (depending on read/write mix)
- **Tail latency degradation**: p99 < 2.5ms under saturation

## Code Quality

Generated code demonstrates:
- ✅ Compile-time type safety (mismatched parameters = compile error)
- ✅ Constexpr metadata for validation
- ✅ Zero-cost abstractions (no virtual dispatch in hot path)
- ✅ Proper arena allocator integration
- ✅ Clean separation: DTOs → validators → handlers → router
- ✅ RFC 7807 Problem Details for all errors
