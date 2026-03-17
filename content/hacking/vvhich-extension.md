---
title: "VVhich Extension?"
date: 2026-03-16T16:59:32+05:30
tags: ["windows", "malware", "rust", "microsoft"]
category: ["hacking"]
---

While preparing for [my talk at Insomni Hack](https://insomnihack.ch/talks/from-code-to-compromise-turning-modern-day-ides-into-attack-vectors-via-malicious-extensions/) and reviewing the [sinister-vsix](https://github.com/whokilleddb/sinister-vsix) project, I wondered: _"Hey, how does VSCode fetch the metadata for these extensions?"_. Just a small recap, during the [Task#4](https://github.com/whokilleddb/sinister-vsix/tree/main/task4) - we spoofed a [Microsoft Published Extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.live-server). 

However, upon examining the extension code, I noticed a couple of things: 
- Metadata like stars, download count, etc wasnt stored in the source code (which made sense)
- _"Where is the blue tick coming from?"_

So, I decided to take a deeper look. This blog documents the result of the research done quickly at an airport while I am on my way to present my fully finished 100+ slides deck(boy oh boy do I have to change those!) - but hopefully it's not _tooo_ incohorent. 

## VVhere is the metadata coming from?

![](https://woforgmedia.wordonfire.org/wp-content/uploads/2024/10/30144433/the-witch.jpg)

First, we go searching for the source of the metadata. A bit of poking around later, I found that VSCode made a POST request to the marketplace API as such:

```bash
curl -X POST "https://marketplace.visualstudio.com/_apis/public/gallery/extensionquery" -H "Content-Type: application/json" -H "Accept: application/json;api-version=7.2-preview" -H "User-Agent: VSCode" -d '{
    "filters": [{
      "criteria": [
        { "filterType": 7, "value": "ms-vscode.live-server" }
      ],
      "pageSize": 1,
      "pageNumber": 1,
      "sortBy": 0,
      "sortOrder": 0
    }],
    "assetTypes": [],
    "flags": 950
  }' 
```

The only thing to note here is the `value` parameter which is set to `ms-vscode.live-server` (this is always set to `extension_publisher:extension_name`). On issuing this request, we get a big response, which (after prettifying) looks like this:

```json
{
  "results": [
    {
      "extensions": [
        {
          "publisher": {
            "publisherId": "5f5636e7-69ed-4afe-b5d6-8d231fb3d3ee",
            "publisherName": "ms-vscode",
            "displayName": "Microsoft",
            "flags": "verified",
            "domain": "https://microsoft.com",
            "isDomainVerified": true
          },
          "extensionId": "4eae7368-ec63-429d-8449-57a7df5e2117",
          "extensionName": "live-server",
          "displayName": "Live Preview",
          "flags": "validated, public, preview",
          "lastUpdated": "2026-02-09T09:27:53.193Z",
          "publishedDate": "2021-06-21T20:33:59.11Z",
          "releaseDate": "2021-06-21T20:33:59.11Z",
          "shortDescription": "Hosts a local server in your workspace for you to preview your webpages on.",
          "versions": [
            {
              "version": "0.5.2026020901",
              "flags": "validated",
              "lastUpdated": "2026-02-09T09:27:53.193Z",
              "files": [
                {
                    <a_bunch_of_stuff>
                }
              ],
              "properties": [
                    <a_bunch_of_github_links>
              ],
              "assetUri": "https://ms-vscode.gallery.vsassets.io/_apis/public/gallery/publisher/ms-vscode/extension/live-server/0.5.2026020901/assetbyname",
              "fallbackAssetUri": "https://ms-vscode.gallerycdn.vsassets.io/extensions/ms-vscode/live-server/0.5.2026020901/1770629024303"
            }
          ],
          "categories": [
            "Other"
          ],
          "tags": ["browser", "html", "live", "livepreview", "preview", "refresh", "reload"],
          "statistics": [
            {
              "statisticName": "install",
              "value": 11966368.0
            },
            {
              "statisticName": "averagerating",
              "value": 4.4358973503112793
            },
            {
              "statisticName": "ratingcount",
              "value": 78.0
            },
            {
              "statisticName": "trendingdaily",
              "value": 0.0022916290304234662
            },
            {
              "statisticName": "trendingmonthly",
              "value": 2.4470581465690677
            },
            {
              "statisticName": "trendingweekly",
              "value": 0.5461268883145507
            },
            {
              "statisticName": "updateCount",
              "value": 19214822.0
            },
            {
              "statisticName": "weightedRating",
              "value": 4.4381077195781415
            },
            {
              "statisticName": "downloadCount",
              "value": 53216.0
            }
          ],
          "deploymentType": 0
        }
      ],
      "pagingToken": null,
      "resultMetadata": [
        {
          "metadataType": "ResultCount",
          "metadataItems": [
            {
              "name": "TotalCount",
              "count": 1
            }
          ]
        }
      ]
    }
  ]
}
```

So that is a bunch of juicy information! So this is how it get's the Metadata. The stats are fetched from the `statistics` key (duh!). We also have a bunch of information like publisher name, publisher id, extension information and more! So that answers our main question about the stats. I am assuming that the blue tick comes from the _domain_ parameter somehow. 

Now, if in the original post request - we saw it only takes two parameters: extension publisher and extension name. So here is the next idea: what if we can control these parameters, which brings me to my next point: How does VSCode fetch these details about an extension?

Let's start with some code.

## Back to coding

First, generate a template javascript extension with:

```bash
$ yo code
```

I decided to name my extension `vvitch-extension`. The inital `package.json` looks like this:

```json
{
  "name": "vvhich-extension",
  "displayName": "vvhich-extension",
  "description": "Demo extension",
  "version": "0.0.1",
  "engines": {
    "vscode": "^1.110.0"
  },
  "categories": [
    "Other"
  ],
  "activationEvents": [],
  "main": "./extension.js",
  "contributes": {
    "commands": [{
      "command": "vvhich-extension.helloWorld",
      "title": "Hello World"
    }]
  },
  "scripts": {
    "lint": "eslint .",
    "pretest": "npm run lint",
    "test": "vscode-test"
  },
  "devDependencies": {
    "@types/vscode": "^1.110.0",
    "@types/mocha": "^10.0.10",
    "@types/node": "22.x",
    "eslint": "^9.39.3",
    "@vscode/test-cli": "^0.0.12",
    "@vscode/test-electron": "^2.5.2"
  }
}
```

Now, let's package this with:

```bash
vsce package --no-yarn --allow-missing-repository --skip-license 
```

This should give us a `vvhich-extension-0.0.1.vsix` extension file, which we can install with:

```
code --install-extension vvhich-extension-0.0.1.vsix     
```

This should install our very basic extension which practically does nothing at this point. But it begs the question - how are these extensions loaded and where is it stored. For this demo, I will be using MacOS - but the technique should be translative to Windows and Linux.

So, back to VSCode. On a clean installation of VSCode, extensions are stored under `~/.vscode/extensions`. Looking at our post-installation state, we see the following:

```
$ ls ~/.vscode/extensions
extensions.json                            undefined_publisher.vvhich-extension-0.0.1
```

The `extensions.json` file looked interesting. Examining it's contents, we see:

```
$ cat ~/.vscode/extensions/extensions.json | jq 
[
{
  "identifier": {
    "id": "undefined_publisher.vvhich-extension"
  },
  "version": "0.0.1",
  "location": {
    "$mid": 1,
    "fsPath": "/Users/db/.vscode/extensions/undefined_publisher.vvhich-extension-0.0.1",
    "external": "file:///Users/db/.vscode/extensions/undefined_publisher.vvhich-extension-0.0.1",
    "path": "/Users/db/.vscode/extensions/undefined_publisher.vvhich-extension-0.0.1",
    "scheme": "file"
  },
  "relativeLocation": "undefined_publisher.vvhich-extension-0.0.1",
  "metadata": {
    "installedTimestamp": 1773697132124,
    "pinned": true,
    "source": "vsix"
  }
}
]
```

_Interesting_

Now, lets uninstall our extension and install the [Live Preview extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.live-server) from the store. Examining the `extensions.json` file after that, we see:

```
$ cat ~/.vscode/extensions/extensions.json | jq                      
[
  {
    "identifier": {
      "id": "ms-vscode.live-server"
    },
    "version": "0.4.17",
    "location": {
      "$mid": 1,
      "fsPath": "/Users/db/.vscode/extensions/ms-vscode.live-server-0.4.17",
      "external": "file:///Users/db/.vscode/extensions/ms-vscode.live-server-0.4.17",
      "path": "/Users/db/.vscode/extensions/ms-vscode.live-server-0.4.17",
      "scheme": "file"
    },
    "relativeLocation": "ms-vscode.live-server-0.4.17",
    "metadata": {
      "installedTimestamp": 1773723790211,
      "pinned": false,
      "source": "gallery",
      "id": "4eae7368-ec63-429d-8449-57a7df5e2117",
      "publisherId": "5f5636e7-69ed-4afe-b5d6-8d231fb3d3ee",
      "publisherDisplayName": "Microsoft",
      "targetPlatform": "undefined",
      "updated": false,
      "private": false,
      "isPreReleaseVersion": false,
      "hasPreReleaseVersion": false
    }
  }
]
```

So, now we can make a few assumption:
- VSCode reads the `extensions.json` to get the `id` of the extension and then uses that to fetch the extension details
- It also reads the some more information from the `metadata` field, namely, the `source` parameter. Remember how in the [sinister-vsix](https://github.com/whokilleddb/sinister-vsix) project we talked about the visual detection where the extension page listed if it has been installed from a VSIX or not? Yeah - we can get rid of that too.

Time to modify our code. 

