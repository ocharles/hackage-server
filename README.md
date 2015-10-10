# hackage-server

[![Build Status](https://travis-ci.org/haskell/hackage-server.png?branch=master)](https://travis-ci.org/haskell/hackage-server)

This is the `hackage-server` code. This is what powers <http://hackage.haskell.org>, and many other private hackage instances.

## Installing ICU

You'll need to do the following to get hackage-server's dependency `text-icu` to build:

### Mac OS X

    brew install icu4c
    brew link icu4c --force

### Ubuntu/Debian

    sudo apt-get update
    sudo apt-get install unzip libicu-dev

## Setting up security infrastructure

If `datafiles/` is your static files directory (containing, for instance,
`datafiles/templates`), you will need to create a directory `datafiles/TUF`. Use
the [hackage-repo-tool](http://hackage.haskell.org/package/hackage-repo-tool) to
create private keys:

    hackage-repo-tool create-keys --keys /path/to/keys

Then copy over the timestamp and snapshot keys to the TUF directory:

    cp /path/to/keys/timestamp/<id>.private datafiles/TUF/timestamp.private
    cp /path/to/keys/snapshot/<id>.private  datafiles/TUF/snapshot.private

Create root information:

    hackage-repo-tool create-root --keys /path/to/keys -o datafiles/TUF/root.json

And finally create a list of mirrors (this is necessary even if you don't have
any mirrors):

    hackage-repo-tool create-mirrors --keys /path/to/keys -o datafiles/TUF/mirrors.json

The `create-mirrors` command takes a list of mirrors as additional arguments if
you do want to list mirrors.

At this point your server is good to go. In order for secure clients to
bootstrap the root security metadata from your server, you will need to provide
them with the public key IDs of your root keys; you can find these as the file
names of the files created in `/path/to/keys/root` (as well as in the generated
root.json under the `signed.roles.root.keyids`). An example `cabal` client
configuration might look something like

    remote-repo my-private-hackage
      url: http://example.com:8080/
      secure: True
      root-keys: 18a11971b3491c697cb46e94141f50f7ee043ddc5bade200744b95543c53771f
                 7b0e2516c2dd2501ca95b82f209fb8b769680ec7ce5aec4e0abab25600222791
                 ed1e79078ce86a8e8dcc32358e10357e156eb42f95385c0c9a7d231e23867676
      key-threshold: 2

NOTE: The `hackage-repo-tool` is rather rudimentary at the moment. Key
management will change before the official release of the Hackage Security
project.

## Running

    cabal install -j --enable-tests

    hackage-server init
    hackage-server run

If you want to run the server directly from the build tree, run

    dist/build/hackage-server/hackage-server run --static-dir=datafiles/

By default the server runs on port `8080` with the following settings:

    URL:      http://localhost:8080/
    username: admin
    password: admin

To specify something different, see `hackage-server init --help` for details.

The server can be stopped by using `Control-C`.

This will save the current state and shutdown cleanly. Running again
will resume with the same state.

### Resetting

To reset everything, kill the server and delete the server state:

```bash
rm -rf state/
```

Note that the `datafiles/` and `state/` directories differ:
`datafiles` is for static html, templates and other files.
The `state` directory holds the database (using `acid-state`
and a separate blob store).

### Creating users & uploading packages

* Admin front-end: <http://localhost:8080/admin>
* List of users: <http://localhost:8080/users/>
* Register new users: <http://localhost:8080/users/register>

Currently there is no restriction on registering, but only an admin
user can grant privileges to registered users e.g. by adding them to
other groups. In particular there are groups:

 * admins `http://localhost:8080/users/admins/` -- administrators can
   do things with user accounts like disabling, deleting, changing
   other groups etc.
 * trustees `http://localhost:8080/packages/trustees/` -- trustees can
   do janitorial work on all packages
 * mirrors `http://localhost:8080/packages/mirrorers/` -- for special
   mirroring clients that are trusted to upload packages
 * per-package maintainer groups
   `http://localhost:8080/package/foo/maintainers` -- users allowed to
   upload packages
 * uploaders `http://localhost:8080/packages/uploaders/` -- for
   uploading new packages

### Mirroring

There is a client program included in the hackage-server package called
hackage-mirror. It's intended to run against two servers, syncing all the
packages from one to the other, e.g. getting all the packages from the old
hackage and uploading them to a local instance of a hackage-server.

To try it out:

1. On the target server, add a user to the mirrorers group via
   http://localhost:8080/packages/mirrorers/
1. Create a config file that contains the source and target
   servers. Assuming you are cloning the packages on
   <http://hackage.haskell.org> locally, you could create a config
   file as follows:

```bash
echo -e "http://hackage.haskell.org\nhttp://admin:admin@localhost:8080/" > servers.cfg
```

1. Run the client, pointing to the config file:

```bash
hackage-mirror servers.cfg
```

This will do a one-time sync, and will bail out at the first sign of
trouble. You can also do more robust and continuous mirroring. Use the
flag `--continuous`. It will sync every 30 minutes (configurable with
`--interval`). In this mode it carries on even when some packages
cannot be mirrored for some reason and remembers them so it doesn't
try them again and again. You can force it to try again by deleting
the state files it mentions.
