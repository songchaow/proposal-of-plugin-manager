Please see [the PR](<https://github.com/musescore/MuseScore/pull/4833>) which contains the latest feature list.
## Plugin Store

**Basic**

- [x] Fetch the list of 3.x packages and display them in the table when the resource manager starts.
- [x] Check whether each package in the table has been installed.
- [ ] Uninstall plugin packages via the buttons.
- [ ] Ability to check whether each installed plugin is up-to-date.
- [ ] Searching and filtering facilities.
- [x] Ability to find GitHub repos within the plugin detail page.
- [ ] Ability to choose the correct branch from one GitHub repo.
- [x] Ability to get the latest commit hash and download address of one particular GitHub repo branch.
- [ ] Ability to get the release ID and download address of the latest 3.x version release of one GitHub repo.
- [ ] (Difficult) Ability to choose a correct attachment link of the latest plugin version within the plugin detail page from musescore.org.
- [ ] (Difficult) Determine whether to download from GitHub repos or attachments if both exist.
- [x] Report errors if there’re no download links or GitHub repos found within the page.
- [x] Download the file via the direct link.
- [ ] Ability to get timestamp of the downloaded file from musescore.org.
- [ ] Check whether the downloaded file points to an archive or a qml.
- [ ] Scan the archive for files to be reserved(qml and translation-related files).
- [x] Extract and copy file(s) to be reserved into a subfolder in MuseScore plugin dir.
- [x] Maintain related data structure and the xml file

**Advanced**

- [ ] Display 2.x packages in the list and mark the difference.
- [ ] Add link entries to the plugin detail page and the issue page for each plugin package somewhere in the table.
- [ ] Run the routine of checking update regularly in the background.
- [ ] Check integrity of each installed plugin.
- [ ] Check if there are qml files in the archive that are duplicated of local existing ones during installing and make corresponding actions.

## Local plugin management

**Basic**

- [ ] Original facilities from the old plugin manager. (They should be ported to the “Installed plugins” tab)
- [ ] Show a tree view of all plugins, with each plugin name as the tree node, and qml files as the node’s children.

**Advanced**

- [ ] Report results of whether one qml file is successfully registered or not.
- [ ] If one 2.x qml file fails to be registered, try converting them to the 3.x version.

