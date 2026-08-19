# ESP Source Strategy

The `repos/` tree is a routed evidence store. Search the selected repo module first; broad searches across all repositories should be a fallback.

## Preferred Search Path

1. User project.
2. Selected repo module source, examples, `Kconfig`, and manifests.
3. Selected repo module `resources/`.
4. Cross-repo indexes such as `resources/recipe_index.md` and `resources/repo_index.md`.

## Reduce Noise

- Avoid treating all `repos/*/SKILL.md` files as public entrypoints in one request.
- Search exact symbols first, then prefixes such as `esp_`, `CONFIG_`, `ESP_ERR_`, `idf_component`, or component-specific namespaces.
- For Kconfig, search both `Kconfig` and generated `sdkconfig` names.
- For examples, prefer the one matching target, feature, and ESP-IDF branch.

## Version Discipline

ESP APIs move. A correct answer should name the assumed ESP-IDF/component version when it affects code. If the local repo, recipe, and user project disagree, follow the user's project version and state what changed.
