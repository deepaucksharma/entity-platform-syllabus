# 📚 New Relic Message Queues Monitoring - Ultra Detailed Product Wiki

## Welcome to the Message Queues Wiki

This comprehensive wiki provides an in-depth technical and functional specification for the New Relic Message Queues monitoring system, specifically designed for Kafka infrastructure monitoring across AWS MSK and Confluent Cloud platforms.

## 📖 Table of Contents

### 🏗️ Architecture & Platform
1. **[System Architecture](./01-system-architecture.md)** - Core architecture, entity platform integration, and technical stack
2. **[NR1 Platform Deep Dive](./02-nr1-platform-deep-dive.md)** - New Relic One SDK, Nerdpack structure, and development framework
3. **[Entity Model & Synthesis](./03-entity-model-synthesis.md)** - Entity definitions, relationships, and synthesis rules

### 📊 Data Layer
4. **[NRQL Query System](./04-nrql-query-system.md)** - Query patterns, optimization strategies, and provider-specific queries
5. **[GraphQL & NerdGraph](./05-graphql-nerdgraph.md)** - Entity search, relationships, and API patterns
6. **[Data Flow & Processing](./06-data-flow-processing.md)** - End-to-end data pipeline and transformation logic

### 🎨 User Interface
7. **[UI Components Library](./07-ui-components-library.md)** - Reusable components, patterns, and design system
8. **[Screen Specifications](./08-screen-specifications.md)** - Detailed screen-by-screen implementation
9. **[Visualization Systems](./09-visualization-systems.md)** - HoneyComb view, charts, and interactive elements

### 🔧 Implementation Details
10. **[State Management](./10-state-management.md)** - Application state, URL state, and data caching
11. **[Filter System](./11-filter-system.md)** - Advanced filtering, search, and query building
12. **[Performance Optimization](./12-performance-optimization.md)** - Query optimization, component optimization, and caching

### 🔌 Integrations
13. **[Provider Integrations](./13-provider-integrations.md)** - AWS MSK and Confluent Cloud specifics
14. **[APM Integration](./14-apm-integration.md)** - Producer/consumer relationships and APM entity linking
15. **[Alert Integration](./15-alert-integration.md)** - Alert policies, severity mapping, and notifications

### 📈 Metrics & Analytics
16. **[Metric Definitions](./16-metric-definitions.md)** - Complete metric catalog and calculations
17. **[Dashboard Configuration](./17-dashboard-configuration.md)** - Mosaic system and chart configurations
18. **[Health Calculations](./18-health-calculations.md)** - Health status algorithms and thresholds

### 🔐 Security & Operations
19. **[Security Model](./19-security-model.md)** - Permissions, access control, and data security
20. **[Deployment & Operations](./20-deployment-operations.md)** - Build, deploy, and monitoring procedures

### 📚 Developer Resources
21. **[Development Guide](./21-development-guide.md)** - Setup, coding standards, and contribution guidelines
22. **[Testing Strategy](./22-testing-strategy.md)** - Unit, integration, and E2E testing approaches
23. **[Extension Guide](./23-extension-guide.md)** - Adding new providers, metrics, and features

### 🔮 Advanced Topics
24. **[Query Utils Deep Dive](./24-query-utils-deep-dive.md)** - Central query system analysis
25. **[Entity Relationships](./25-entity-relationships.md)** - Complex relationship modeling
26. **[Future Roadmap](./26-future-roadmap.md)** - Planned enhancements and architecture evolution

## 🚀 Quick Start

### For Users
- Start with [Screen Specifications](./08-screen-specifications.md) to understand the UI
- Review [Metric Definitions](./16-metric-definitions.md) for available metrics
- Check [Provider Integrations](./13-provider-integrations.md) for setup instructions

### For Developers
- Begin with [System Architecture](./01-system-architecture.md) for technical overview
- Study [NR1 Platform Deep Dive](./02-nr1-platform-deep-dive.md) for framework understanding
- Reference [Development Guide](./21-development-guide.md) for coding practices

### For Operators
- Review [Deployment & Operations](./20-deployment-operations.md) for deployment procedures
- Check [Security Model](./19-security-model.md) for access control
- Understand [Health Calculations](./18-health-calculations.md) for monitoring logic

## 📋 Version Information

- **Wiki Version**: 1.0.0
- **Application Version**: Based on SDK v3
- **Last Updated**: January 2024
- **Maintainer**: New Relic APM Team

## 🤝 Contributing

This wiki is maintained alongside the codebase. To contribute:
1. Submit PRs to update documentation with code changes
2. Follow the established format and structure
3. Include examples and diagrams where helpful
4. Keep technical accuracy as the top priority

## 📞 Support

- **Internal Teams**: Contact the APM team via Slack #message-queues
- **External Users**: File issues in the GitHub repository
- **Documentation Issues**: Submit PRs to improve this wiki

---

*This wiki represents the complete technical and functional specification for the New Relic Message Queues monitoring system.*