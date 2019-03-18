## Design Overview

A new app-store like Plugin Manager is planned to be implemented. This tool can reside in Resource Manager(find it in MuseScore menu: Preferences->General->Update translations, or Help->Resource Manager) as a new tab, or just replace the old plugin manager.

In short, two main facilities are covered in this new plugin manager:

- (Part I) Automatically install and manage plugin packages from [MuseScore plugin repository](https://musescore.org/en/plugins).

- (Part II) Traditional facilities for local installed plugins(some of those plugins may be not published online), 

### Part I

Conceptual UI demo I made by merely modifying the UI file:

![ui demo](ui_plugin_store.png)

All available plugins from the online plugin repository will be displayed in this table. 

**Possible features:**

- you can search for new plugins and filter them by category labels.

- you can download new plugins and install them simply by clicking the "Install" button.

  > For plugins that are incompatible for your MuseScore version, the manager should disable the install button.

- For downloaded plugins, you can enable, disable or delete them by clicking corresponding buttons.

### Part II

Conceptual UI demo I made by merely modifying the UI file:

![ui demo](ui_local_plugin.png)

All installed plugins, including plugins downloaded from repository and qmls you manually added, will be displayed in the left QListWidget.

**Possible features:**

- You can still manually download the qml file yourself or write your own plugin qmls locally, and copy them to plugin directory. Then they will appear after reloading.

- Plugins downloaded from repository will also be displayed.

  > Some plugin packages from repository contain multiple qml files. Therefore multiple entries may be displayed for a single plugin package.

- If local plugins are checked to be incompatible, necessary prompts can be added. 

- The traditional plugin manager's facilities will be reserved in this tab, including shortcut configuration, displaying name, path, version and description.

  For display of plugin name, I recommend **using name from title from plugin page**, **not the base filename**.

### Miscellaneous

- To make things consistent, Add "Check for new version of MuseScore plugins" in MuseScore->Preferences -> Update.

  And add corresponding facility of update reminder.

  

## Implementation

### Fetch from Web

When we launch the resource manager, the crawler should fetch a list of available plugins from `https://musescore.org/en/plugins?category=All&compatibility=some_version_id`.

The map between MuseScore version and`some_version_id` is:

| MuseScore Version | `some_version_id` |
| ----------------- | ----------------- |
| 1.x               | 4311              |
| 2.x               | 4316              |
| 3.x               | 4321              |

> It's not hard to parse the raw HTML table using some libraries, though the parsing code would be simpler if there were a JSON file.

Each entry of the plugin list should contain:

- Title(string)
- API Compatibility(a tuple of `bool` indicating compatibilities for 3 versions)
- URL of the plugin page(string)
- Categories(`vector` of enum object)

> **Naming Issues**
>
> There are two type of strings that are related to the identity of one plugin.
>
> The first is the title in the plugin page, and the second is the qml filename that appears in MuseScore menu.
>
> The filename can only be known after we've downloaded the package. Furthermore, for a single plugin package, there can be multiple qml files, leading to multiple menu items.
>
> Therefore, I suggest using the title in the page for displaying in the "Resource Manager" widget. However, names that appear as menu items in Plugin's drop-down menu can keep their original filename form.

### Download

Then when we click on `Download` button of any entry of plugin, the crawler will fetch the plugin page and extract structured data, including:

- GitHub repo URL(if applicable)
- Attachments that are recognized to be suitable plugin distributions

> This fetch procedure can also be considered to run automagically in advance and store in cache, as the total number of plugins is not huge.

If GitHub repo URL is available, we should use [Github Release APIs](https://developer.github.com/v3/repos/releases) to check if there's a release. 

Else, look for direct links in the page or the attachments.

These steps would require sophisticated pattern recognizing algorithms to choose the correct version(2.x or 3.x, etc. ), which can be further discussed and optimized later.

### Extract and Install

There are various types of downloaded plugin files. Some are zips, and others are just qmls.

> There are [some code snippets](https://github.com/musescore/MuseScore/blob/1d5ae8afbb4b83b36558c1e365e8794d170d5065/mscore/resourceManager.cpp#L291) used for unzipping language packages, which can be used similarly for plugin packages.

After the zips are extracted, we should remove irrelevant files such as README and others, and copy all qml files and folders that contain them to the plugin directory.

Warnings should be reported if there are conflicts when copying files, such as qml files with the same name already exist.

### Maintaining the Local Plugin List

downloaded plugins(belongs to one plugin package), and local plugins, should be both maintained.

plugins.xml stored in disk.

`QList<PluginDescription> _pluginList` is maintained by `PluginManager`.

[Keep map between package metadata and filenames!]

[Alter plugins.xml to add multiple xmls in one plugin]

[Add plugin version]

On the other hand, scan for all qml files locally in `void PluginManager::loadList(bool forceRefresh)` should be done.

### Compatibility Check

Each plugin will have its API compatibility specified in its web page.

As far as I know, plugins of compatibility 1.x, 2.x and 3.x can only be run on MuseScore 1.x, 2.x and 3.x respectively. Some of 2.x plugins can be ported to 3.x with some code changes.

Compatibility check should happen in two cases:

- When installing plugins from repository, before the download begins, compatibility list specified in the plugin web page should be verified against current MuseScore version.

- When importing/reloading local plugins, manually check whether the plugin is imported successfully.

  (this can be done by analyzing the result of `QQmlComponent::create()`. See [example code](https://github.com/musescore/MuseScore/blob/1d5ae8afbb4b83b36558c1e365e8794d170d5065/mscore/plugin/mscorePlugins.cpp#L91), where the `errors()` method contains related info.)

### Automatic Update

Most plugins don't have their own version numbers currently. So how to detect updates of plugins seems to be a tough problem. 

Another stupid way is to download the whole plugin again and look for difference.