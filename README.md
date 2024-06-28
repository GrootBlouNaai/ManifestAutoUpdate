# Steam Manifest Repository

## Project Overview

* Automatically crawl `Steam` game manifests using `Actions`

## Project Structure

* `main` branch
    * `main.py`: Main program for crawling manifests
        * `-c, --credential-location`: Path to store account credentials, default is `data/client`
        * `-l, --level`: Log level, default is `INFO`
        * `-p, --pool-num`: Number of accounts to crawl simultaneously, default is `8`
        * `-r, --retry-num`: Number of retries for failures or timeouts, default is `3`
        * `-t, --update-wait-time`: Interval time for re-crawling accounts, in seconds, default is `86400`
        * `-k, --key`: Key for decrypting `users.json`
            * Required if re-cloning after pushing to remote or running with `Actions`
            * Manual decryption: Save the key to `KEY` file, install `git-crypt`, switch to `data` branch and run `git-crypt unlock KEY`
        * `-i, --init-only`: Only initialize, does not crawl manifests
        * `-C, --cli`: Enter interactive login if login fails
        * `-P, --no-push`: Prevent automatic push after crawling
        * `-u, --update`: Determine accounts to crawl by fetching all app information from the repository
        * `-a, --app-id`: Limit crawling to specified app IDs, multiple IDs can be specified, separated by spaces
        * `-U, --users`: Limit crawling to specified accounts, multiple accounts can be specified, separated by spaces
    * `storage.py`: Import manifests into the repository
        * `-r, --repo`: Specify repository
        * `-a, --app-id`: Game ID
        * `-p, --app-path`: Directory in the repository's app branch format
    * `apps.py`: Export all game information from the repository to `apps.xlsx`
        * `-r, --repo`: Specify repository
        * `-o, --output`: Save directory
    * `merge.py`: Automatically merge `pr` for `Actions`
        * `-t, --token`: Personal access token
        * `-l, --level`: Log level, default is `INFO`
    * `push.py`: Push branches
    * `pr.py`: Create pull requests for branches
        * `-r, --repo`: Specify repository
        * `-t, --token`: Personal access token
* `data` branch: Used for storing account data, automatically checked out to `data` directory after first run initialization
    * `data/client`: Directory for storing account credential files and `cm` server information, place account `ssfn` files here
    * `data/users.json`: Stores account and password
        * Format: `{"account": ["password", "ssfnxxxx"], "account": ["password", null], ...}`
        * Fill `null` if no `ssfn`
    * `data/appinfo.json`: Stores `appid` corresponding to `manifest id`
        * Format: `{"11111": "manifest id", ...}`
    * `data/userinfo.json`: Stores account's owned `appid` information and whether it is disabled, etc.
        * Format: `{"account": {"app": [11111, 22222, ...], "update": 1673018145, "enable": true, "status": 63}, ...}`
            * `update`: Last update timestamp
            * `enable`: Whether it is disabled
            * `status`: Reason for login failure - [EResult](https://partner.steamgames.com/doc/api/steam_api#EResult)
    * `data/.gitattributes`: Records files to be encrypted by `git-crypt`
        * Default encryption: `users.json client/*.key 2fa.json`
    * `data/2fa.json`: Records account `2fa` information
        * Format: `{"account": "shared_secret", ...}`
* Branches named after `appid`: Used for storing manifests and key files
    * `depots/xxx`: If the `app` has new manifests after program run, it will pull the corresponding `appid` branch from remote, or create an empty `appid` branch using the first commit of `main` branch, and check it out to `depots/corresponding appid branch` directory, e.g., `depots/11111`
        * `depots/xxx/repository_id_manifest_id.manifest`: Manifest file
        * `config.vdf`: Key file, refer to `Steam/config/config.vdf` for format
            * ```vdf
              "depots"
              {
                  "repository_id"
                  {
                      "DecryptionKey" "repository_key"
                  }
              }
              ```
* `tag`: Marks each manifest commit
    * Naming format: `repository_id_manifest_id`
    * Used for filtering already crawled manifests

## Running Process

1. `.github/workflows/CI.yml`
    * Use `Actions` to periodically crawl manifests
2. Enable multi-threading to log in and crawl manifests for multiple accounts simultaneously until all accounts are crawled
    * Check if the account is disabled
    * Check if the time since the last crawl is greater than the crawl interval
    * Fetch all crawlable manifests for the account, use `tag` to filter already crawled manifests
3. After crawling, call `push.py` to upload `branches` and `tags`, and push `data` branch

## How to Deploy

1. Fork this repository (skip the following steps if initializing with `Actions`)
2. Install git and configure your `github` account
3. Clone your forked repository
    * `git clone https://github.com/your_name/ManifestAutoUpdate --recurse-submodules --depth=1`
        * `--recurse-submodules`: Clone submodules
        * `--depth=1`: Shallow clone
4. Install dependencies
    * `pip install -r requirements.txt`
5. Run the program
    * `python main.py`
6. Initialize
    * The first run of the program will perform initialization
    * Initialization will generate the `data` branch and check it out to the `data` directory using `worktree`
    * Generate a key for encrypting `users.json`
        * The key generation path is: `data/KEY`
        * The program will also output the hexadecimal string of the key, which needs to be stored in the github repository secret, named `KEY`
            * Open your repository -> `Settings` -> `Secrets` -> `Actions` -> `New repository secret`
            * Or add `/settings/secrets/actions/new` to your repository URL
    * Add account passwords to `data/users.json`:
        * If you need to use `Actions` later, push it to the remote repository
            * Re-run the program, it will automatically push to the `data` branch at the end of the program
            * Manual push steps:
                1. `cd data`: Switch to `data` directory
                2. `git add -u`: Add modified content
                3. `git commit -m "update"`: Commit changes
                4. `git push origin data`: Push to remote `data` branch
7. Initialize and run with Actions
    * Configure `workflow` read and write permissions: Repository -> `Settings` -> `Actions` -> `General` -> `Workflow permissions` -> `Read and write permissions`
    * Open `Actions` in the repository, select the corresponding `Workflow`, click `Run workflow`, and select parameters to run
        * `INIT`: Initialize
            * `users`: Accounts, multiple accounts can be specified, separated by commas
            * `password`: Passwords, multiple passwords can be specified, separated by commas
            * `ssfn`: [ssfn](https://ssfnbox.com/), upload this file to `credential_location` directory in advance, multiple files can be specified, separated by commas
            * `2fa`: [shared_secret](https://zhuanlan.zhihu.com/p/28257212), multiple secrets can be specified, separated by commas
            * `update`: Whether to update accounts
            * `update_users`: Accounts to be updated
            * After the first initialization, remember to save the key to the repository secret, otherwise it will report an error next time due to the lack of a key, and then remember to delete the results of this `Workflow` run to prevent key leakage, or use local initialization for more security
        * `CI`: Crawl all accounts
        * `PR`: Automatically `pr` manifests to the specified repository
            * Since `Github` prohibits `Actions` [recursively creating pr](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#triggering-a-workflow-from-a-workflow), you need to create a [personal access token](https://github.com/settings/tokens/new) and save it to the repository secret `GITHUB_TOKEN`
            * `repo`: Repository address
        * `MERGE`: Automatically check `pr` and merge manifests
        * `UPDATE`: Add `-u` parameter

## How to PR Manifests

* This project uses `Actions` to periodically check and merge manifests, check if the merge is successful after `Actions` run

1. Complete the deployment of this project and crawl manifests
2. Open the branch you want to `pr` manifests to, click `Compare & pull request`
3. Click `Create pull request` to create a `pr`

## Telegram Discussion Group

* [SteamManifestShare](https://t.me/SteamManifestShare)

## Repository Game View

1. [apps.xlsx](https://github.com/wxy1343/ManifestAutoUpdate/raw/data/apps.xlsx)
2. [Online View](https://docs.google.com/spreadsheets/d/1tS-Tar11TAqnlaeh4c7kHJq-vHF8QiQ-EtcEy5NO8a8)
