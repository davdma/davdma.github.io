# Note on How to Modify the Site

After the hassle of trying to start the blog on my personal website, I think it's important to document how
to navigate and edit the repo beyond what is provided in [CUSTOMIZE doc](CUSTOMIZE.md).

# Removing or Adding Functionality

To remove a functionality without needing to delete files, simply add to the `exclude` list on line 170 of `_config.yml`. More instructions in the customize doc. To add it back in just remove those mentions from the `exclude` list.

# Editing the Blog

- Simply create markdown files in `_posts` directory, with formats provided by al-folio blog examples.
- The blog page clickable tags and categories can be modified in the `_config.yml` file lines 262-263.
- Can keep drafts in the `_drafts` folder.

# Working with Prettier and Lychee

The Prettier tests and link checking (Lychee) tests fail often are annoying. To resolve that,
make sure to call `npx prettier . --write` in the repo to format. Use `npx prettier . --check` to
run the tests if you do not want to immediately format.

The Lychee tests can be ignored for certain links by adding to the `.lycheeignore` file in the repo.
