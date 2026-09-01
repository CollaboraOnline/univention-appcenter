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

Download the `univention-appcenter-control` tool (it needs Python 3.7+
and curl):

    curl -O https://provider-portal.software-univention.de/appcenter-selfservice/univention-appcenter-control
    chmod +x univention-appcenter-control

The tool asks for a username and password: use the credentials of your
account at https://provider-portal.software-univention.de/.

Create the new app version as a copy of the latest published one, then
upload the files:

    ./univention-appcenter-control new-version 5.2/collabora=25.04.9.4 5.2/collabora=26.04.3.2
    cd 5.2/collabora
    ../../univention-appcenter-control upload 5.2/collabora=26.04.3.2 \
        ini settings env configure_host inst preinst uinst test \
        README_EN README_DE README_APPLIANCE_EN README_APPLIANCE_DE \
        univention-config-registry-variables

and likewise with `5.2/collabora-online=<app version>` for the Collabora
Online app.

For unattended use, store the password in a file (without a trailing
newline) and pass the credentials on the command line:

    printf '%s' 'PASSWORD' > ~/.appcenter-pwd
    chmod 600 ~/.appcenter-pwd
    ./univention-appcenter-control upload --username USERNAME \
        --pwdfile ~/.appcenter-pwd --noninteractive \
        5.2/collabora=26.04.3.2 ini settings

Uploads go to the Test App Center; the release to the production App
Center is requested through the Provider Portal. The upload tool can only
add or replace files - to delete one (such as the dropped in-container
configure script), clear the corresponding field in the Provider Portal.

## Testing from the Test App Center

On a UCS test machine, switch the App Center to the Test App Center and
install the app:

    univention-install univention-appcenter-dev
    univention-app dev-use-test-appcenter
    univention-app update
    univention-app install collabora

After uploading changed app files, refresh the local cache and exercise
the change, e.g.:

    univention-app update
    univention-app configure collabora --set username=admin --set password=secret

A settings change recreates the container; the result can be checked in
the container environment:

    docker inspect "$(ucr get appcenter/apps/collabora/container)" \
        --format '{{range .Config.Env}}{{println .}}{{end}}'

`univention-app shell` does not work with these apps (the distroless
image has no shell). `univention-app dev-use-test-appcenter --revert`
switches the machine back to the production App Center.

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
