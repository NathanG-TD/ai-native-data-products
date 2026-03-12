# AI-Native Data Product Design Standards

A comprehensive, integrated architectural framework for building AI-Native Data Products on Teradata that enable autonomous agent discovery and operation.

---

## Overview

This repository contains a complete set of design standards for building data products optimized for AI agent consumption. Unlike fragmented approaches that treat vector stores, feature stores, and knowledge graphs as independent silos, these standards provide a unified framework where six specialized modules work together as an integrated system.

The Semantic module serves as a universal discovery map, enabling agents to autonomously navigate the complete data product structure through queryable metadata—eliminating the need for pre-programmed system knowledge or manual integration of disconnected components.

---

## What's Included

### Core Design Standards (6 Modules)

- **Domain Module** - Core business entities with bi-temporal tracking (single source of truth)
- **Semantic Module** - Comprehensive metadata layer enabling agent discovery of all modules
- **Prediction Module** - Feature engineering and ML prediction storage with point-in-time correctness
- **Search Module** - Vector embeddings for semantic similarity search using Teradata native VECTOR
- **Memory Module** - Agent learning, session state, and discovered patterns
- **Observability Module** - Event tracking, data quality monitoring, and lineage (OpenLineage aligned)

### Supporting Documentation

- **Master Design Standard** - Complete architectural blueprint with physical naming conventions
- **Advocated Data Management Standards** - Implementation guidance and best practices
- **Agent Bootstrap Prompt Fragment** - Reusable agent initialization protocol

### Skills

- **Teradata JSON Skill** - Working with JSON data in Teradata
- **Teradata Recursive SQL Skill** - Graph traversal and multi-hop queries
- **AI-Native Module Design Skills** - Domain, Semantic, Prediction, Search, Memory, Observability

---

## Key Features

### Integrated Architecture
All six modules designed to work together with standard integration patterns, zero data duplication, and consistent temporal tracking across modules.

### Agent Discovery Protocol
Three-tier discovery hierarchy enables agents to autonomously discover module locations, table structures, and relationships from a single entry point.

### Multi-Hop Relationship Discovery
Recursive SQL patterns find join paths between any tables across modules.

### Platform-Native Design
Leverages Teradata capabilities: native VECTOR datatype, massively parallel processing, co-located joins, and JSON handling.

### Open Standards Alignment
Integrates with OpenLineage (data lineage), ODCS (data contracts), and Feast patterns (feature stores).

### Comprehensive Metadata
Over 410 COMMENT statements demonstrate self-describing design—every table and column documented for both human and agent understanding.

---

## Getting Started

### 1. Understand the Architecture

Review **AI_Native_Data_Product_Master_Design.md** for the architectural blueprint, including:
- Six module descriptions
- Integration patterns
- Physical naming conventions
- Agent discovery system

### 2. Implement Modules Incrementally

**Recommended deployment order**:
1. **Domain** - Foundation (business entities)
2. **Semantic** - Discovery layer (metadata and relationships)
3. **Prediction/Search** - Enhancement layers (features and vectors)
4. **Observability** - Monitoring layer (quality and events)
5. **Memory** - Learning layer (agent state)

### 3. Review Module-Specific Standards

Each module has a detailed design standard document:
- Domain_Module_Design_Standard.md
- Semantic_Module_Design_Standard.md
- Prediction_Module_Design_Standard.md
- Search_Module_Design_Standard.md
- Memory_Module_Design_Standard.md
- Observability_Module_Design_Standard.md

### 4. Enable Agent Discovery

Use **Agent_Bootstrap_Prompt_Fragment.md** to configure agents for autonomous navigation.

---

## Use Case Examples

These design standards can be applied to build:

- **Customer 360** - Unified customer view with behavioral features and embeddings
- **Fraud Detection** - Real-time transaction analysis with historical patterns
- **Product Recommendations** - Semantic search with ML-powered suggestions
- **Risk Analytics** - Predictive risk scoring with audit trails
- **Knowledge Management** - Document search with relationship discovery
- **Any AI-driven data product** requiring integrated analytics, ML, and search

---

## Design Principles

### 1. Modularity First
Each module is independently deployable and composable—start with Domain and Semantic, add others as needed.

### 2. Zero Data Duplication
All modules reference Domain entities via foreign keys. Use views to join modules for complete context—maintain single source of truth.

### 3. Self-Describing
Comprehensive metadata in Semantic module enables autonomous agent discovery without human guidance.

### 4. Agent-Native Design
Optimized for machine interpretation through queryable metadata tables, standard patterns, and multi-hop relationship discovery.

### 5. Standards-Driven
Aligns with open standards (OpenLineage, ODCS, Feast) for interoperability and best practices.

### 6. "Big Questions, Small Answers"
Agents ask SQL questions, the database processes millions of records; metadata modules store table-level references and aggregate metrics.

---

## Technical Highlights

### Tested on Teradata
- Semantic module multi-hop path discovery validated
- Memory module VARCHAR table references validated
- All SQL syntax verified on Teradata Community Edition v20.0

### Comprehensive Metadata
- 410+ COMMENT statements on tables and columns
- Every table self-documenting
- Demonstrates metadata best practices

### Platform Optimization
- Native VECTOR datatype for embeddings
- Recursive CTEs for path discovery
- JSON handling with dot notation
- Co-located joins for performance

---

## Documentation

All design standards include:
- Table definitions with comprehensive COMMENT statements
- Integration patterns and examples
- Query patterns and SQL examples
- Design checklists and quality criteria
- Cross-references to other modules

**Total**: ~200 pages of production-ready design documentation

---

## Contributing

This is a Teradata-maintained design standard. For questions, suggestions, or feedback:

- Open an issue in this repository
- Contact: [Your contact information]

---

## License

Copyright © 2025 Teradata Corporation

Licensed under Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0)

See [LICENSE.md](LICENSE.md) for full terms.

---

## Acknowledgments

Developed by Teradata's Worldwide Data Architecture Team, Field Technology Organization.

---

**Version**: 1.0  
**Last Updated**: February 2026  
**Status**: Production Ready
