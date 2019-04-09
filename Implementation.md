This chapter describes the internal workflow of a plugin’s installation procedure. Intended readers are **MuseScore developers**.

See [here](README.md) for all documents related to this project.

## 1.1      Fetching from the MuseScore Plugin Repository

When we launch the resource manager, the crawler should fetch a list of available plugin packages from https://musescore.org/en/plugins?category=All&compatibility=some_version_id.

The map between MuseScore version and `some_version_id` is:

| MuseScore Version | some_version_id |
| ----------------- | --------------- |
| 1.x               | 4311            |
| 2.x               | 4316            |
| 3.x               | 4321            |

Only 3.x plugin packages are fetched by default.

It's not hard to parse the raw HTML table using some libraries, though the parsing code would be simpler if there were a JSON file.

Each entry of the plugin list should contain:

- Title(string)

  The title here should use the title from the plugin detail page, not the name of qml files.

- API Compatibility(a tuple of bool indicating compatibilities for the 3 versions)

- URL of the plugin detail page(string)

- Categories(vector of enum object)

The class `PluginPackageMeta` in the [draft implementation code](https://github.com/musescore/MuseScore/blob/96196532ca68ca6ded2f009ac9c2da0113b891b1/mscore/plugin/pluginManager.h#L33) reflects this structure.

## 1.2      Analyzing the plugin detail page and download link

Then when we click on the “Install” button of any entry of plugin, the crawler will fetch the plugin detail page.

The plugin may be stored either from GitHub or attachments, both of which should be checked.

### 1.2.1        From GitHub

If GitHub repo links are found in the detail page’s HTML, further determine whether to download from a release or from a branch,

Usually, GitHub releases are believed to be more reliable, so we should always choose the release if there is one. [Github Release APIs](https://developer.github.com/v3/repos/releases) should be used to further fetch the direct download address and the release ID.

We should also make sure that we are downloading the latest release of  **3.x version**. This could be checked via the name of the repo branch from which the release is published. The info is available in the "target_commitish" field in the returned JSON of GitHub APIs. See [here](https://api.github.com/repos/jeetee/MuseScore_TempoChanges/releases) for an example of a returned JSON.

If GitHub releases don't exist, download from a suitable branch that corresponds to the latest version. The process of determining the correct branch seems to be easy: many plugin repos have branches named master, 2.x or 3.x. A plausible approach is to select a branch that contains "3.x", or select "master" if there are not branches that contain “3.x”.

### 1.2.2        From attachments

Look for direct links in the plugin detail page or attachments from musescore.org.

The attachments in the page could have various names and various kinds of description around them.

Therefore, this step would require sophisticated pattern recognizing algorithms to choose download links of the correct version(2.x or 3.x, etc.), which is quite time-consuming and could be further discussed and optimized later.

### 1.2.3        Determine whether to download from GitHub or attachments

Maybe GitHub repos are supposed to be more preferred since they keep version info like commit history and release IDs, which could be used for checking update.

But sometimes for some plugins, we do need to download from musescore.org, since the GitHub repo contains a wrong version(1.x or 2.x ones). See these plugin detail pages for example:

<https://musescore.org/en/project/check-parallel-fifths-and-octaves>

<https://musescore.org/en/project/check-harmony-rules>

So the logic to select between GitHub or attachments should be added and refined later.

 

Finally, a structured description of this plugin package will be recorded, including:

- GitHub repo URL(if applicable).

- URLs that are recognized to be downloadable links for proper plugin archives or qml file.

- latest commit hash of the corresponding branch on GitHub if applicable. This field is used for checking updates.

- latest release ID of GitHub releases if applicable. This field is used for checking updates.

The class `PluginPackageDescription` in the[ draft implementation code](https://github.com/musescore/MuseScore/blob/96196532ca68ca6ded2f009ac9c2da0113b891b1/mscore/plugin/pluginManager.h#L54) reflects this structure.

## 1.3      Download

Download via the link analyzed above.

When downloading files from musescore.org,  `Last Modified` field from the HTTP response should also be added to the description of the package. This field is used for checking updates.

## 1.4      Extracting and Installing

### 1.4.1        Determine the file extension name

First, we need to check whether we've downloaded a qml file or an archive.

If the file is downloaded from GitHub, we will always get an archive of that repository. If from musescore.com, the extension could always be checked in the string suffix of the direct link.

The manager should extract the downloaded file if it's an archive, or directly install if it's a qml file.

### 1.4.2        Get the file list and determine the files to be reserved

There are [some code snippets](https://github.com/musescore/MuseScore/blob/1d5ae8afbb4b83b36558c1e365e8794d170d5065/mscore/resourceManager.cpp#L291) in MuseScore’s codebase that are already used for unzipping language packages, which could be used similarly for plugin packages.

Before the archive is extracted, we could use the class `MQZipReader` in MuseScore to read the file list contained in the archive.

Following file types are necessary to reserve:

●     Plugin files(*.qml)

●     Translation related files(*.qm and *.ts)

The path of each qml file could be recorded in PluginPackageDescription. It's helpful in checking the plugin’s integrity.

### 1.4.3        Check duplicated qml files

Before copying, the chosen qml files are compared against each plugin in the _pluginList variable in class PluginManager. If a local plugin with the same name is found, ask the user for options described in Part I: Plugin Store.

### 1.4.4        Extract the files selected and copy to the plugin directory

Finally, the qml files and translation files are copied into a new folder under MuseScore’s plugin directory. The name of the new folder could be the last URL slug of the plugin detail page. For example: `tunings-and-temperaments` for <https://musescore.org/en/project/tunings-and-temperaments>.

## 1.5      Maintaining Local Plugins and Plugin Packages

### 1.5.1        Permanent Storage

Currently, MuseScore stores metadata for local plugins in plugins.xml(located in C:\[your username]\AppData\Local\MuseScore\MuseScore3). The XML file only stores the qml file paths and flags indicating whether each qml file is loaded. After MuseScore is launched, these items will be read into `QList<PluginDescription> _pluginList` in class `PluginManager`. (See the process [here](https://github.com/musescore/MuseScore/blob/a9df4a02c07cf5666644be620c6b951becedade8/mscore/plugin/pluginManager.cpp#L67))

Qml files of each plugin **from store** should also be added into plugins.xml. Those matters are taken good care of by existing plugin related codes and don't need modification.

But beyond these, for each plugin package installed from the repository, additional metadata should be maintained. Essentially, all members in PluginPackageDescription, plus the plugin detail page URL, should be serialized and stored.

- The plugin detail page’s URL is needed for the manager to mark the corresponding item in the table as “Installed”. (We cannot tell that merely from installed qml files.)

- Other members in PluginPackageDescription are stored mainly for checking updates. See the section "Automatic Update".

The above metadata could be saved in a separate xml file(currently saved as pluginpackages.xml in the [draft implementation](https://github.com/musescore/MuseScore/blob/cdc3b74c0c057e4d7be3055452d1e3e7d91bf9c2/mscore/resourceManager.cpp#L682), in the same directory as plugins.xml).

### 1.5.2        Runtime Data Structure

Currently MuseScore uses `QList<PluginDescription> PluginManager::_pluginList` to maintain local plugins. When MuseScore launches, it fills this QList with contents from plugins.xml, loads and registers plugins that are marked to load.

For plugins from the store, the following classes are added:

```c++
struct PluginPackageMeta;

enum PluginPackageSource;

struct PluginPackageDescription;
```

The classes `PluginPackageMeta` and `PluginPackageDescription` are described in previous chapters. And `PluginPackageSource` is just an enum class specifying the source is GitHub, GitHub release or attachment.

In runtime, a `PluginPackageMeta` object is created for every plugin displayed in the table, even if it hasn’t been installed; and a `PluginPackageDescription` object is created for each installed plugin from the store.

The objects of `PluginPackageDescription` are deserialized from pluginpackages.xml when the resource manager starts.

## 1.6      Compatibility Check

Each plugin has its API compatibility specified on its web page. Plugins of compatibility 1.x, 2.x and 3.x can only be run on MuseScore 1.x, 2.x and 3.x respectively.

Compatibility check should happen in two cases:

- When installing plugins from the repository, before the download begins, compatibility list specified in the plugin web page should be verified against current MuseScore version. Normally 2.x plugin packages won't be shown in the list.

- When registering local plugins, check whether the plugin is registered successfully.

Currently, registrations of incompatible plugins will silently fail, and no menu buttons will be displayed for those plugins. This could be improved by reporting whether the plugin is compatible explicitly.

The result of registration is contained in the return value of QQmlComponent::create(). See [example code](https://github.com/musescore/MuseScore/blob/1d5ae8afbb4b83b36558c1e365e8794d170d5065/mscore/plugin/mscorePlugins.cpp#L91), where the errors() method contains related info.

### 1.6.1        Try Converting the 2.x Plugin

Some of 2.x plugins could be converted to 3.x ones with small code changes. The plugin manager could try doing this job by applying the following two replacements in qml files:

- from `import MuseScore 1.0` to `import MuseScore 3.0`

- from `import FileIO 1.0` to `import FileIO 3.0`

The effect may be limited, but might works sometimes.

The conversion could take place if the plugin failed to be registered.

## 1.7      Automatic Update

Most plugins don't have their own version numbers currently.

However, there are other ways to check if a particular plugin package has been changed:

- If the download link has changed in the plugin detail page, it's reasonable to assume there's an update.

- For plugin files stored in musescore.com, the Last Modified field of HTTP Response header is available to check.

  When checking updates, we could send HTTP HEAD requests to the direct download links of those plugins, If newer Last Modified value is found in response, there's probably an update.

- For plugins stored in GitHub, the commit history could be checked via[ Github Release APIs](https://developer.github.com/v3/repos/releases). If newer commit logs are found, there's probably an update.

Another stupid way is to download the whole plugin again and look for difference, which is not likely to be adopted.