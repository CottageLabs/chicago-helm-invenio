# University of Chicago InvenioRDM Initial Deployment Procedure

## 1. Ensure secrets are correctly set up.

| Secret                  | Custom | Variable                       | Note                                        |
|-------------------------|--------|--------------------------------|---------------------------------------------|
| invenio                 | No     | INVENIO_CSRF_SECRET_SALT       |                                             |
|                         |        | INVENIO_SECRET_KEY             |                                             |
|                         |        | INVENIO_SECURITY_CHANGE_SALT   |                                             |
|                         |        | INVENIO_SECURITY_CONFIRM_SALT  |                                             |
|                         |        | INVENIO_SECURITY_LOGIN_SALT    |                                             |
|                         |        | INVENIO_SECURITY_PASSWORD_SALT |                                             |
|                         |        | INVENIO_SECURITY_REMEMBER_SALT |                                             |
|                         |        | INVENIO_SECURITY_RESET_SALT    |                                             |
| invenio-basic-auth      | Yes    | htpasswd                       | This is a temporary measure pre-production. |
| invenio-datacite-secret | Yes    | DATACITE_USERNAME              |                                             |
|                         |        | DATACITE_PASSWORD              |                                             |
| invenio-db-secret       | Yes    | password                       |                                             |
| invenio-mail-secret     | Yes    | MAIL_DEFAULT_SENDER            |                                             |
|                         |        | MAIL_PASSWORD                  |                                             |
|                         |        | MAIL_PORT                      |                                             |
|                         |        | MAIL_SERVER                    |                                             |
|                         |        | MAIL_USERNAME                  |                                             |
|                         |        | MAIL_USE_TLS                   |                                             |
| invenio-mq-secret       | Yes    | password                       |                                             |
| invenio-oauth-secret    | Yes    | CHI_OAUTH_CLIENT_ID            |                                             |
|                         |        | CHI_OAUTH_CLIENT_SECRET        |                                             |
| invenio-rabbitmq        |        | rabbitmq-erlang-cookie         |                                             |
| invenio-rabbitmq-config |        | rabbitmq.conf                  |                                             |
| invenio-sysadmin-secret | No     | password                       | Password for a super admin.                 |

We need to take a backup of the salts.

## 2. Verify environment variables in `values-uchicago.yaml`

hostname 
init needs to be true

Make sure datacite is correctly set up, with testmode false.

## 3. Set up import user

This will be `knowledge_admin@uchicago.edu`. This user needs to be admin temporarily, in order to
create the communities.

```bash
invenio roles create community-curator

invenio roles create record-curator

invenio users create knowledge_admin@uchicago.edu --password <password> --active --confirm

invenio roles add knowledge_admin@uchicago.edu record-curator

invenio roles add knowledge_admin@uchicago.edu community-curator

invenio roles add knowledge_admin@uchicago.edu admin
```

Some accounts need to be given `record-curator` roles. Note that they need to have been created first.

## 4. Import vocabularies

```bash
invenio vocabularies update --vocabulary funders --origin ror-http

invenio vocabularies update --vocabulary affiliations --origin ror-http
```

## 5. Copy data into bucket

Copy the

* XML metadata file
* content file folder
* file information csv
* file download statistics csv
* checksum XML file?

onto the S3 bucket 

Log on to the console and check the data is present in `/mnt/import-data`

## 5. Import data

Run the import script.

```bash
./scripts/console.sh --namespace invenio

invenio shell \
	site/chicago_invenio/scripts/import_data.py \
	knowledge_admin@uchicago.edu \
	<data_file> \ # e.g. /mnt/import-data/export.xml
	<root file folder> \ #  e.g. /mnt/import-data/files
	<path for restricted files> & #  e.g. /mnt/import-data/restricted
```

The import script will output a log file, an `errors.json` with information about the records that couldn't
be imported as well as `doi_mapping.csv` which contains record identifiers against DOIs that have the prefix
defined in `values-uchicago.yaml`.

Remove admin role from import user?

## 6. Import usage statistics

```bash
invenio shell \ 
	site/chicago_invenio/scripts/migrate_statistics.py \
	<file_information.csv>
	<file_statistics.csv>
```

This will populate the search indices. Might need to run a re-index?

## 7. Set up DOIs

Using `doi_mapping.csv` which was outputted from the import script, run an ad hoc script to determine the
Invenio PIDs, and use that information to update Datacite records?

## 8. Verify checksums

If we are given an XML file with checksums.

```bash
invenio shell \ 
	site/chicago_invenio/scripts/verify_checksums.py \
	<root file folder> \ #  e.g. /mnt/import-data/files
	<checksums.xml>
```