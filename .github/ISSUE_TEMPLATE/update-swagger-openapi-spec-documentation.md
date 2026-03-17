---
name: Update Swagger OpenApi-Spec documentation
about: Update Swagger OpenApi-Spec documentation
title: Update Swagger OpenApi-Spec documentation pt
labels: documentation
assignees: Copilot
type: Task

---

Below Pull-Request in the Backend-Repository changes some REST-API relevant stuff:
https://github.com/aura-historia/backend/pull/
I want you to update the OpenApi-Spec with all the changes introduced in this Pull-Request.
Remove old types, add new ones or adjust existing ones.
Especially, carefully check which types changed and fix them!
We need 100% precision here. I want you to verify your changes multiple times.

Moreover I want you to document the changes in our Changelog.md.
This is not for api-versioning (think api/v1/, api/v2, ...) but for internal communication between front- and backend.

NB: You will need to inspect the changes in the code to determine both - the updated OpenApi Spec and the changelog. Go through every line!
