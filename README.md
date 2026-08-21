# tf-module-qdrant

Terraform module that deploys Qdrant, a performant vector store for managing vectors and supporting Vector RAG.

This is a reusable module. Another Terraform project calls it and supplies the inputs.

## Usage

```hcl
module "qdrant" {
  source = "git::https://github.com/yggdrasil-analytics/tf-module-qdrant.git"
}
```

Pin the source with `?ref=<tag>` so callers get a fixed version rather than whatever is on `main`.
