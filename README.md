# Collabora Online apps for the Univention App Center

This repository holds the app metadata and configuration scripts for the
two Collabora apps published in the
[Univention App Center](https://www.univention.com/products/app-catalog/):

- `collabora` - Collabora Online Development Edition (CODE)
- `collabora-online` - Collabora Online

The apps deploy the respective docker images on Univention Corporate
Server (UCS).

The App Provider Portal is the publishing channel, but this repository is
the source of truth: edit the files here, then upload them to the Test App
Center with Univention's `univention-appcenter-control` tool.

## Layout

Each app's files live in a directory named after the UCS version and the
app id, mirroring the arguments that the upload tool takes, e.g.

    5.0/collabora/          CODE app files
        ini                 app metadata
        env                 container environment (admin credentials etc.)
        settings            App Settings definitions
        ...
    5.0/collabora-online/   Collabora Online app files
        ini
        env
        settings
        ...

## Uploading to the Test App Center

Download `univention-appcenter-control` from the
[App Provider Portal](https://provider-portal.software-univention.de/)
(requires Python 3.7+ and curl), then:

    ./univention-appcenter-control upload --username "$USER" \
        5.0/collabora=<app version> ini env settings

and likewise with 5.0/collabora-online=<app version> for the Collabora
Online app.

Add `--pwdfile <file> --non-interactive` for unattended use. Uploads go to
the Test App Center; the release to the production App Center is requested
through the Provider Portal.

## Configuration notes

The Collabora Online docker image is distroless: it contains no shell, so
App Center scripts cannot run inside the container. All configuration is
passed as container environment variables, which coolwsd reads at startup
(`--use-env-vars`): `username` and `password` for the admin console,
`extra_params` for arbitrary `--o:` config overrides, `aliasgroupN`,
`dictionaries` and `DONT_GEN_SSL_CERT`.

## License

This repository is licensed under the GNU Affero General Public License,
version 3. See [LICENSE](LICENSE).
