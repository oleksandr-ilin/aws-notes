Overview
Enterprise search service with ML-driven relevance and connectors to many data sources.

Tasks
- Configure indexes, data sources, relevance tuning, and query logging.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| Elasticsearch/OpenSearch | More control | More ops overhead |
| Custom search + embeddings | Tailored relevance | Engineering effort |

Limitations
- Connector coverage and feature limits; tuning required for domain-specific relevance.

Price info
- Per-query + capacity pricing depending on index size and traffic.

Network & multi-region
- Regional service; replicate indexes or centralize search depending on needs.

Popular use cases
- Document search across intranets, knowledge bases, and customer support content.

When Not to Use
- Completely custom ranking algorithms or extreme scale without capacity planning.

DR strategy
- Backup index snapshots and export knowledge bases; keep index configuration in IaC.
