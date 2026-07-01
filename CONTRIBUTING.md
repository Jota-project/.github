# Contributing to Jota 🚀

First off, thank you for considering contributing to Jota!

### Estado de los repos

Antes de abrir un PR, mira el [`README.md`](README.md) raíz y [`ARCHITECTURE.md`](ARCHITECTURE.md) para entender qué servicios están `Maintained`, `Alternative` o `Deprecated`. Los PRs van primero contra los repos `Maintained`.

### 🧪 Development Workflow

1. **Docker is King:** Every service has a `Dockerfile`. Ensure your changes don't break the build:
   `docker build -t jota-service-check .`
2. **Linting:**
   * Python: use `ruff` or `black`.
   * C++: follow the existing Clang-Format style.
3. **Tests:** Run tests before submitting a PR.

### 📬 Pull Request Process

* Create a feature branch (`feat/your-feature`).
* Include tests for new logic.
* Update the relevant `README.md` if parameters change.
* For doc-only changes, one PR per repo is the standard.

### 📚 Documentación

Las decisiones arquitectónicas grandes van en `ARCHITECTURE.md`. Cambios pequeños van en el README del repo específico. Specs de diseño van en `docs/superpowers/specs/`.
