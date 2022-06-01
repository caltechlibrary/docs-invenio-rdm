# Upgrading from v8.0 to v9.0

## Prerequisites

The steps listed in this article require an existing local installation of InvenioRDM 8.0, please make sure that this is given!

If unsure, run `invenio-cli install` from inside the instance directory before executing the listed steps.

!!! warning "Backup"

    Always backup your database and files before you try to perform an upgrade.

## Upgrade Steps

!!! warning "Upgrade your invenio-cli"

    Make sure you have the latest `invenio-cli`, for InvenioRDM v9 the release is v1.0.4

### Installing the Latest Versions

Clean-up some old packages and bump the InvenioRDM version:

```bash
# Uninstall modules that were forked/removed and might cause namespace issues
pipenv run pip uninstall -y invenio-iiif flask-security-invenio flask-security flask-kvsession flask-kvsession-invenio

# Upgrade to InvenioRDM v9
invenio-cli packages update 9.0.1
```

These commands should take care of locking the dependencies for v9 and installing them.

### Updating Configuration Variables

To enable the new communities feature you have to remove from your config or set `COMMUNITIES_ENABLED` to `True`.

If you are using password-based login you have to add this configuration variable for backwards compatibility:

```python
SECURITY_PASSWORD_SINGLE_HASH = ['pbkdf2_sha512']
```

### Update styling

Because of a refactoring to make styling and theming easier, you need to update the following files:

`{{project_shortname}}/assets/less/site/globals/site.overrides`

``` diff
- @import "@less/invenio_app_rdm/theme";
```

`{{project_shortname}}/assets/less/site/globals/site.variables`

```diff
- @import "@less/invenio_app_rdm/variables";
```

`{{project_shortname}}/assets/less/theme.config`

```diff
/* Path to theme packages */
- @themesFolder : 'themes';
+ @themesFolder : '~semantic-ui-less/themes';

...

@siteFolder : '../../less/site';
+ @imagesFolder : '../../images';

...

/*******************************
         Import Theme
*******************************/

- @import (multiple) "~semantic-ui-less/theme.less";
+ @import (multiple) "themes/rdm/theme.less";
```

You can afterwards build the instance assets:

```bash
invenio-cli assets build
```

### Data Migration

You first need to first fix an issue with the database versioning table:

```bash
# Change entry in alembic versions table
pipenv run invenio shell -c "from invenio_db import db; db.session.execute(\"UPDATE alembic_version SET version_num ='f701a32e6fbe' WHERE version_num='f261e5ee8743'\"); db.session.commit()"
```

Now you can run the database schema upgrade, add new fixtures and perform the records data migration:
```bash
# Perform the database migration
pipenv run invenio alembic upgrade

# Add new fixtures
pipenv run invenio rdm-records fixtures

# Run data migration script
pipenv run invenio shell $(find $(pipenv --venv)/lib/*/site-packages/invenio_app_rdm -name migrate_8_0_to_9_0.py)
```

### Elasticsearch

The last required step is the migration of Elasticsearch indices, to add all the new communities features.

```bash
pipenv run invenio index delete "communitymembers-*" --yes-i-know
pipenv run invenio index delete "request_events-*" --yes-i-know
pipenv run invenio index destroy --yes-i-know
pipenv run invenio index init
pipenv run invenio rdm-records rebuild-index
```

This will ensure that all indices and their contents are based on the latest definitions and not out of date.

As soon as the indices have been rebuilt, the entire migration is complete! :partying_face: