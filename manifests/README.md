# manifests/

This folder is where **you** save the YAML you write as you work through the phases. It starts
mostly empty on purpose — writing the manifests yourself is the point.

Suggested layout as you progress:

```
manifests/
├── web-deployment.yaml         # Phase 4
├── web-service.yaml            # Phase 4
├── demo-configmap.yaml         # Phase 5
├── demo-pvc.yaml               # Phase 5
├── demo-ingress.yaml           # Phase 6
├── demo-app/                   # Phase 8 (multi-service dress rehearsal)
│   ├── redis.yaml
│   ├── api.yaml
│   └── ingress.yaml
└── mcp-gateway/                # Phase 9 (the capstone)
    ├── configmap.yaml
    ├── secret.yaml             # DO NOT COMMIT REAL SECRETS (see .gitignore)
    ├── mongodb.yaml
    ├── auth.yaml
    ├── registry.yaml
    ├── gateway.yaml
    └── ingress.yaml
```

The exact filenames don't matter — consistency does. Keeping every manifest here means Claude Code
can see your whole cluster's desired state in one place and reason about it with you.

**Never commit real secret values.** The repo's `.gitignore` excludes common secret patterns, but
double-check before pushing a public repo.
