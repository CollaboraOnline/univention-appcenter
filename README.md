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

    5.2/collabora/          CODE app files
        ini                 app metadata
        settings            App Settings definitions
        configure_host      runs on the docker host after settings change
        inst                join script (reverse proxy, integrations)
        ...
    5.2/collabora-online/   Collabora Online app files
        ini
        settings
        configure_host
        env                 container environment template
        inst
        ...

## Uploading to the Test App Center

Download `univention-appcenter-control` from the
[App Provider Portal](https://provider-portal.software-univention.de/)
(requires Python 3.7+ and curl), then:

    ./univention-appcenter-control upload --username "$USER" \
        5.2/collabora=<app version> ini settings configure_host inst ...

and likewise with `5.2/collabora-online=<app version>` for the Collabora
Online app.

Add `--pwdfile <file> --non-interactive` for unattended use. Uploads go to
the Test App Center; the release to the production App Center is requested
through the Provider Portal.

## Configuration notes

Since 26.04 the Collabora Online docker image is distroless: it contains no
shell, so App Center scripts cannot run inside the container (the apps ship
no `configure` script and `DockerScriptConfigure` is empty). All
configuration is passed as container environment variables, which coolwsd
reads at startup (`--use-env-vars`): `username` and `password` for the
admin console, `extra_params` for arbitrary `--o:` config overrides,
`aliasgroupN`, `dictionaries`, `server_name`, `remoteconfigurl`,
`content_security_policy`, `DONT_GEN_SSL_CERT` and `cert_domain`.

The App Center writes every inside-scope app setting into the env file it
passes to `docker create`, both under its own name and uppercased - so the
`username` and `password` settings reach coolwsd directly. The app `env`
file maps the remaining settings: `domain` to `aliasgroup1`, and the
support key (Collabora Online app only) to
`extra_params=--o:support_key=...`. The support key is uploaded as a key
file (written to `/etc/collabora.key` on the host); `configure_host` syncs
its content into the UCR variable `collabora/online/supportkey` that the
env file references.

Docker fixes a container's environment at creation time, so each app ships
a `configure_host` script that runs `univention-app reinitialize` after a
settings change; reinitialize recreates the container with the current
settings values. `univention-app shell` does not work with these images
(there is no shell to run).

## License

This repository is licensed under the GNU Affero General Public License,
version 3. See [LICENSE](LICENSE).
