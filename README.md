# Gas Town Specifications

Agent specifications for Gas Town

## Archteypes

Each archetype represents a character with a specified role:
- **pm**: Rig coordinator, domain knowledge, project context
- **auteur**: UI/UX, components, styling, layout, interactivity
- **penny**: Security, bugs, dead code, test coverage
- **bob**: Clean code, naming, structure, refactoring

Drop the archetype markdown into your crew member's directory within the rig.

### PM

The PM is basically a rig-level Mayor. He accepts tasks from the Mayor and Overseer, then delegates them to polecats or crew members. He's the second go-to for domain knowledge, next to the Overseer. Other agents may reach out to him with questions. 

### Auteur

Auteur is the resident UI/UX expert. He guides all design decisions made for the project. He writes all the HTML, CSS, and physics behaviors related to page elements and user interactions.

Fill out the `design-questionnaire.md` and have an agent create a `design-direction.md` from it. Add both of these files to your project `/docs`.

### Penny

Penny is the QA expert. He looks for vulnerabilities, bugs, and unit testing gaps. He is also very concerned with test quality. Most of his work will only ever be in the test directory.

### Bob

Bob (inspired by "Uncle Bob") is the clean code specialist. He is responsible for auditing the code, refactoring, and improving overall code quality.

## Docs

Copy and paste these docs directly into the root of your project. Some docs are language-specific – promote the `language.md` you want as the `directory-name.md` under the `/docs` folder.
