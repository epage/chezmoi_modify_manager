# Examples - ignore and transform flags

Here are some useful examples of flags for various settings files I have come across.

## KDE

### dolphinrc
```bash
ignore section "MainWindow"
ignore section "KPropertiesDialog"
ignore "General" "ViewPropsTimestamp"
ignore "Open-with settings" "History"
```

### kdeglobals
```bash
ignore "General" "ColorSchemeHash"
ignore "KFileDialog Settings" "Show hidden files"
ignore "KFileDialog Settings" "Show Inline Previews"
ignore section ""DirSelect Dialog""
```

### kglobalshortcutsrc
There are two issues in this configuration.

First, ActivityManager switch-to-activity entries. There are multiple entries,
making it a perfect fit for a regular expression. Note that this is not state
per se. It does however seem to vary between computers, having different UUID
values.

Second, certain shortcut keys like flipping between two representations. A
specialised transform has been added to handle this case. When this is needed
you will see diffs like the following:

```diff
-playmedia=none,,Play media playback
+playmedia=none,none,Play media playback
```

In summary, the following seems to work well:

```bash
ignore regex "ActivityManager" "'switch-to-activity-.*'"
transform regex ".*" ".*" kde_shortcut
```

### konversationrc
Konversation has two relevant quirks:

1. It saves the password in the settings file (instead of using kwallet)
2. It resorts it alias list every time.

```bash
ignore "ServerListDialog" "Size"
transform "Aliases" "AliasList" unsorted_list separator=","
transform "Identity 0" "Password" keyring service="konversation" user="konversation_id0"
```

To store the password for Identity 0 in your keyring of choice you can use the
`secret-tool` program from `libsecret` (`libsecret-tools` on Debian/Ubuntu):

```console
$ secret-tool store --label="Konversation password" service konversation username konversation_id0
[Enter your password at the prompt]
```

***Caution!*** Remember to also remove the password from the .src.ini that was
added to the chezmoi directory. Using an [add hook](#add-hook) can help with
this.

### kwinrc
Similar to kglobalshortcutsrc there are computer specific UUIDs.

```bash
ignore regex "Desktops" "'Id_.*'"
```

### plasmanotifyrc

```bash
ignore section "DoNotDisturb"
```

### Trolltech.conf

This is a Qt config, rather than a KDE config (strictly speaking) but since KDE
uses Qt, it is sitll relevant.

```bash
ignore "Qt" "filedialog"
```

## PrusaSlicer / SuperSlicer

PrusaSlicer and the fork SuperSlicer also use INI style files:

### PrusaSlicer.ini / SuperSlicer.ini

```bash
ignore "<NO_SECTION>" "auto_toolbar_size"
ignore "<NO_SECTION>" "freecad_path"
ignore "<NO_SECTION>" "last_output_path_removable"
ignore "<NO_SECTION>" "last_output_path"
ignore "<NO_SECTION>" "version_online_seen"
ignore "<NO_SECTION>" "version_online"
ignore "<NO_SECTION>" "version_system_info_sent"
ignore "<NO_SECTION>" "version"
ignore "<NO_SECTION>" "window_mainframe"
ignore "font" "active_font"
ignore "presets" "filament"
ignore "presets" "print"
ignore "presets" "sla_material"
ignore "presets" "sla_print"
ignore regex "<NO_SECTION>" "desktop_integration_.*"
ignore regex "font:.*" ".*"
ignore regex "presets" "filament_.*"
ignore section "recent_projects"
ignore section "recent"
```

### PrusaSlicerGcodeViewer.ini / SuperSlicerGcodeViewer.ini

```bash
ignore "<NO_SECTION>" "version"
ignore "<NO_SECTION>" "window_mainframe"
ignore section "recent_projects"
```

## KeePassXC

### keepassxc.ini

KeePassXC stores private and public keys for KeeShare in the config.
You may not want to commit this to the repository.

```bash
ignore "KeeShare" "Active"
ignore "KeeShare" "Foreign"
ignore "KeeShare" "Own"
```

## GTK-3.0/GTK-4.0

### settings.ini

The file `~/.config/gtk-<version>/settings.ini` has a DPI value in it that
changes between computers. Thus, each of those setting files need the
following:

```bash
ignore "Settings" "gtk-xft-dpi"
```

# Examples - hook scripts

## Add hook

A user defined hook script can optionally be executed by chezmoi_ini_add to
filter the data when adding it. This can be useful when readding files to
automatically remove passwords that are managed by a transform.

The hook script should be an executable file in the root of the chezmoi
directory and must be named `.chezmoi_modify_manager.add_hook`.

Here is an example that will filter out passwords of the `konversationrc` file:

```zsh
#!/bin/zsh
# The file from the target directory will be available on STDIN.
# The data to add to the source state should be printed to STDOUT.

# Currently only "ini"
type=$1
# Path of file as provided by the user to the command, may be a relative path
target_path=$2
# Path in the source state we are writing to. Will end in .src.ini for ini files.
source_data_path=$3

if [[ $source_data_path =~ konversationrc ]]; then
    # Filter out any set password.
    sed '/Password=./s/=.*$/=PLACEHOLDER/'
else
    # Let other files through as they are without changes
    cat
fi
```
