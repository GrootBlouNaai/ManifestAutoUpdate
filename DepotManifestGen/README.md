# Steam Depot Manifest File Generation

## Parameters

* `-u, --username`: Account username
* `-p, --password`: Account password
* `-a, --app-id`: Only crawl the specified `appid`
* `-l, --list-apps`: Whether to only print app information
* `-s, --sentry-path, --ssfn`: Path to the `ssfn` file
* `-k, --login-key`: Login key
* `-f, --two-factor-code`: `2fa` authentication code
* `-A, --auth-code`: Email authentication code
* `-i, --login-id`: Login ID
* `-c, --cli`: Interactive login
* `-L, --level`: Log level, default is `INFO`
* `-C, --credential-location`: Path to store account credentials, default is `client`
* `-r, --remove-old`: Whether to delete old manifests after fetching new ones

## Introduction to Manifest Files

* `appid`: Game ID
* `depot`: Depot for storing game files
* `depot_id`: Depot number, usually an incremental number of the `appid`, one `appid` can have multiple `depot_id`s, such as `dlc`, `language`, etc.
* `manifest`: Record of each depot's file list
* `manifest_gid`: Manifest number, similar to `commit id`
* `DecryptionKey`: Depot key used to decrypt the depot manifest file
* For more details, see `https://steamdb.info/app/{app_id}/depots/`

## Location of Manifest Files

* `Steam\depotcache`

## Purpose of Manifest Files

* Used for downloading Steam games
* Reference project [DepotDownloader](https://github.com/SteamRE/DepotDownloader)

## Manifest File Generation

* Dependent project [steam](https://github.com/ValvePython/steam)

```python
from steam.protobufs.content_manifest_pb2 import ContentManifestSignature

# Get manifest_code
manifest_code = cdn.get_manifest_request_code(app_id, depot_id, manifest_gid)
# Get manifest object via manifest_code
manifest = cdn.get_manifest(app_id, depot_id, manifest_gid, decrypt=False, manifest_request_code=manifest_code)
# Get DecryptionKey
DecryptionKey = cdn.get_depot_key(manifest.app_id, manifest.depot_id)
# Decrypt manifest with DecryptionKey
manifest.decrypt_filenames(DecryptionKey)
# Clear signature
manifest.signature = ContentManifestSignature()
for mapping in manifest.payload.mappings:
    # Remove special characters at the end of the filename
    mapping.filename = mapping.filename.rstrip('\x00 \n\t')
    # Sort chunks by sha
    mapping.chunks.sort(key=lambda x: x.sha)
# Sort filenames
manifest.payload.mappings.sort(key=lambda x: x.filename.lower())
# Calculate crc_clear via payload
manifest.metadata.crc_clear = crc32(manifest.payload.size + manifest.payload)
```

* `crc_clear` Calculation
    * ~~After reverse engineering Steam, the algorithm for calculating `crc_clear` was found, the specific code is in `calc_crc_clear.c`~~
    * ~~Analysis showed that Steam performs `crc` calculation on the `ContentManifestPayload` part, the specific process was not understood, only the assembly code was copied~~
    * The length and data of `ContentManifestPayload` are packed and then calculated using `crc32`
    * Reference [Manifest CRC Generation](https://cs.rin.ru/forum/viewtopic.php?t=124734)

## Downloading Games After Importing Manifest Files into Steam

* Copy the generated `.manifest` file to the `Steam\depotcache` directory
* Merge the `depots` from the generated `config.vdf` file into the `Steam\config\config.vdf` file
* Use tools like [steamtools](https://steamtools.net/) to unlock the game, after which it can be downloaded normally
