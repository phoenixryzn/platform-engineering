
```
platform-engineering/
│
├── workloads/
│   └── otel-platform/
│       ├── src/
│       ├── docker/
│       └── tests/
│
├── platform/
│   ├── helm/
│   │    └── otel-platform/
│   │         ├── Chart.yaml
│   │         ├── values.yaml
│   │         ├── values-dev.yaml
│   │         ├── values-prod.yaml
│   │         └── templates/
│   │
│   ├── observability/
│   └── security/
│
├── infrastructure/
│   └── terraform/
│
├── ci-cd/
│   └── github-actions/
│
├── environments/
│   ├── dev/
│   ├── stage/
│   └── prod/
│
└── docs/
```