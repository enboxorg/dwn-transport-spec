# DWN Transport Protocol Specification

### Specification

Latest Draft: https://dwn-transport-spec.pages.dev

### Companion Specifications

- **Decentralized Web Node (Core)**: https://dwn-spec.pages.dev (`enboxorg/dwn-spec`)
- **DWN Sync Protocol**: https://dwn-sync-spec.pages.dev (`enboxorg/dwn-sync-spec`)
- **Enbox Connect Protocol**: https://web5-connect-spec.pages.dev (`enboxorg/web5-connect-spec`)

### Implementation

| Implementer | Language   | Repo                                                                                                            | Environments     |
| ----------- | ---------- | --------------------------------------------------------------------------------------------------------------- | ---------------- |
| Enbox       | TypeScript | [enboxorg/enbox](https://github.com/enboxorg/enbox) (`packages/dwn-server`, `packages/dwn-clients`, `packages/local-node`) | Server, Web, CLI |

### Development

This spec is built using [Spec-Up](https://github.com/nickreserved/spec-up), a tool for generating W3C-style spec documents from Markdown.

```bash
npm install
npm run edit    # live-reload dev server
npm run render  # one-shot build
```

The rendered spec is output to `spec/index.html`.
