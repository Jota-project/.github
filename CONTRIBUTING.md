# Contributing to Jota 🚀

First off, thank you for considering contributing to Jota!

### 🧪 Development Workflow
1. **Docker is King:** Every service has a `Dockerfile`. Ensure your changes don't break the build:
   `docker build -t jota-service-check .`
2. **Linting:** * Python: Use `ruff` or `black`.
   * C++: Follow the existing Clang-Format style.
3. **Tests:** Run tests before submitting a PR.

### 📬 Pull Request Process
* Create a feature branch (`feat/your-feature`).
* Include tests for new logic.
* Update the relevant `README.md` if parameters change.