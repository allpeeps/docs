 ## Overview

The day after a GHES version's [deprecation date](https://github.com/github/docs-internal/tree/main/lib/enterprise-dates.json), a banner on the docs will say: `This version was deprecated on <date>.` This is all users need to know. However, we don't want to update those docs anymore or link to them in the nav.  Follow the steps in this issue to **archive** the docs.

**Note**: Do each step below in a separate PR. Only move on to the next step when the previous PR has been merged.

## Step 0: Remove deprecated version numbers from docs-content issue forms

**Note**: This step can be performed independently of all other steps, and can be done several days before or along with the other steps. 

- [ ] In the `docs-content` repo, remove the deprecated GHES version number from the "Specific GHES version(s)" section in the following files (in the `.github/ISSUE_TEMPLATE/` directory): [`release-tier-1-or-2-tracking.yml`](https://github.com/github/docs-content/blob/main/.github/ISSUE_TEMPLATE/release-tier-1-or-2-tracking.yml) and [`release-tier-3-or-tier-4.yml`](https://github.com/github/docs-content/blob/main/.github/ISSUE_TEMPLATE/release-tier-3-or-tier-4.yml).
- [ ] When the PR is approved, merge it in. This can be merged independently from all other steps. 

## Step 1: Scrape the docs and archive the files

- [ ] In your checkout of the [repo with archived GHES content](https://github.com/github/help-docs-archived-enterprise-versions), create a new branch: `git checkout -b deprecate-<version>`
- [ ] In your `docs-internal` checkout, download the static files for the oldest supported version into your archival checkout:
    The archive script depends on an optional dependency so install optional dependencies first:
    ```
    $ npm ci --include-optional
    ```
    Then run the archive script:
    ```
    $ script/enterprise-server-deprecations/archive-version.js -p <path-to-archive-repo-checkout>
    ```
    If your checkouts live in the same directory, this command would be:
    ```
    $ script/enterprise-server-deprecations/archive-version.js -p ../help-docs-archived-enterprise-versions
    ```
    **Note:** You can pass the `--dry-run` flag to scrape only the first 10 pages plus their redirects for testing purposes.
  
## Step 2: Upload the assets directory to Azure storage

- [ ] Log in to the Azure portal from Okta. Navigate to the [githubdocs Azure Storage Blob resource](https://portal.azure.com/#@githubazure.onmicrosoft.com/resource/subscriptions/fa6134a7-f27e-4972-8e9f-0cedffa328f1/resourceGroups/docs-production/providers/Microsoft.Storage/storageAccounts/githubdocs/overview).
- [ ] Click the "Open in Explorer" button to the right of search box.If you haven't already, click the download link to download "Microsoft Azure Storage Explorer." To login to the app, click the plug icon in the left sidebar and click the option to "add an azure account." When you login, you'll need a yubikey to authenticate through Okta.
- [ ] From the Microsoft Azure Storage Explorer app, select the `githubdocs` storage account resource and navigate to the `github-images` blob container. 
- [ ] Click "Upload" and select "Upload folder." Click the "Selected folder" input to navigate to the `help-docs-archived-enterprise-versions` repository and select the `assets` directory for the version you just generated. In the "Destination folder" input, add the version number. For example, `/enterprise/2.22/`.
- [ ] Check the log to ensure all files were uploaded successfully.
- [ ] Remove the `assets` directory from your `help-docsc-archived-enterprise-versions` repository, we don't want to commit that directory in the next step.

## Step 3: Commit and push changes to help-docs-archived-enterprise-versions repo

- [ ] Search for `site-search-input` in the compressed Javascript files (should find the file in the `_next` directory). When you find it, use something like https://beautifier.io/ to reformat it to be readable. Find `site-search-input` in the file, the result will be enclosed in a function that looks something like... `1125: function () { ... },` Delete the innards of this function, but leave the `function() {}` part.
- [ ] Copy, paste, and save the updated file back into your local `help-docs-archived-enterprise-versions` repository.
- [ ] In your archival checkout, `git add <version>`, commit, and push.
- [ ] Open a PR and merge it in. Note that the version will _not_ be deprecated on the docs site until you do the next step.

## Step 4: Deprecate the version in docs-internal

In your `docs-internal` checkout:
- [ ] Create a new branch: `git checkout -b deprecate-<version>`.
- [ ] Edit `lib/enterprise-server-releases.js` by removing the version number to be deprecated from the `supported` array and move it to the `deprecated` array.
- [ ] Open a new PR. Make sure to check the following:
    - [ ] Tests are passing (you may need to include the changes in step 6 to get tests to pass).
    - [ ] The deprecated version renders in preview as expected. You should be able to navigate to `docs.github.com/enterprise/<DEPRECATED VERSION>` to access the docs. You should also be able to navigate to a page that is available in the deprecated version and change the version in the URL to the deprecated version, to test redirects.
    - [ ] The new oldest supported version renders on staging as expected. You should see a banner on the top of every page for the oldest supported version that notes when the version will be deprecated.
    
## Step 5: Remove static files for the version

- [ ] In your `docs-internal` checkout, create a new branch `remove-<version>-static-files` branch: `git checkout -b remove-<version>-static-files` (you can branch off of `main` or from your `deprecate-<version>` branch, up to you).
- [ ] Run `script/enterprise-server-deprecations/remove-static-files.js` and commit results.
- [ ] Run `script/enterprise-server-deprecations/remove-redirects.js` and commit results.
- [ ] Open a new PR.
- [ ] Get a review from docs-engineering and merge. This step can be merged independently from step 6. The purpose of splitting up steps 5 and 6 is to focus the review on specific files.

## Step 6: Remove the liquid conditionals and content for the version

- [ ] In your `docs-internal` checkout, create a new branch `remove-<version>-markup` branch: `git checkout -b remove-<version>-markup` (you can branch off of `main` or from your `deprecate-<version>` branch, up to you).
- [ ] Remove the outdated Liquid markup and frontmatter.
    - [ ] Run the script: `script/enterprise-server-deprecations/remove-version-markup.js --release <number>`.
    - [ ] Spot check a few changes. Content, frontmatter, and data files should all have been updated.
    - [ ] Open a PR with the results. The diff may be large and complex, so make sure to get a review from `@github/docs-content`.
    - [ ] Debug any test failures or unexpected results -- it's very likely manual updates will be necessary, the script does a lot of work but doesn't automate everything and can't 100% replace human intent.
- [ ] When the PR is approved, merge it in to complete the deprecation. This can be merged independently from step 5. 

## Step 7: Deprecate the OpenAPI description in `github/github`
    
- [ ] In `github/github`, edit the release's config file in `app/api/description/config/releases/`, and change `deprecated: false` to `deprecated: true`.
- [ ] Open a new PR, and get the required code owner approvals. A docs-content team member can approve it for the docs team.
- [ ] When the PR is approved, [deploy the `github/github` PR](https://thehub.github.com/engineering/devops/deployment/deploying-dotcom/).  If you haven't deployed a `github/github` PR before, work with someone that has -- the process isn't too involved depending on how you deploy, but there are a lot of details that can potentially be confusing as you can see from the documentation.

**Note**: you can do this step independently of the other steps after a GHES version is deprecated since it should no longer get updates in github/github.  You should plan to get this PR merged as soon as possible, otherwise if you wait too long our OpenAPI automation may re-add the static files that you removed in step 5.
