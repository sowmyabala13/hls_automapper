# reference/

Put the source's reference material here before you start the build:

- File specs and data dictionaries from the source
- Mapping rules or instructions (naming conventions, renames, required output forms, date formats)
- Notes on your target model, if its tables lack column comments

Genie Code reads this folder with `@` context. Rules here take priority over anything Genie Code infers.

Do **not** put data files here. Data files go in the volume, under the ingestion root.
