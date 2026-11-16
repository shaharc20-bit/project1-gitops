# charts/

Helm charts used by the ApplicationSets.

- `flask-app/` - the Flask AWS Monitor chart. **Copy of `helm/flask-app` in the code repo `project1finel`**; Jenkins refreshes it on every build, do not edit it here.
- `stress-app/` - a small load generator (stress-ng) to test resource limits and the HPA per environment.
