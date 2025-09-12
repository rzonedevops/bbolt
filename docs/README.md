# bbolt Technical Architecture Documentation

This directory contains comprehensive technical architecture documentation for bbolt, a pure Go embedded key/value database.

## Documentation Structure

- **[System Architecture](architecture.md)** - High-level system overview and component relationships
- **[Core Components](components.md)** - Detailed documentation of core components (DB, Bucket, Transaction, etc.)
- **[Transaction Management](transactions.md)** - Transaction lifecycle, MVCC, and concurrency control
- **[Storage Engine](storage.md)** - B+tree structure, page management, and disk layout
- **[Memory Management](memory.md)** - Memory mapping, buffer management, and page allocation
- **[API Operations](api-flows.md)** - Detailed flow diagrams for key operations

## Quick Reference

For a quick overview of the system architecture, start with the [System Architecture](architecture.md) document. Each document includes both textual explanations and Mermaid diagrams for visual understanding.

## Audience

This documentation is intended for:
- Contributors to the bbolt project
- Developers integrating bbolt into their applications who need deep understanding
- Database enthusiasts studying embedded database implementation
- Anyone interested in B+tree and MVCC implementation details

## Related Documentation

- **[User Guide](../README.md)** - Main README with usage examples and API documentation
- **[Package Documentation](../doc.go)** - Go package-level documentation
- **[Code Comments](../)** - Inline code documentation throughout the source