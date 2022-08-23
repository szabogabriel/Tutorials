This page is a backup copy of https://unix.stackexchange.com/questions/103213/how-can-i-add-an-application-to-the-gnome-application-menu

In GNOME and other freedesktop.org-compliant desktop environments, such as KDE and Unity, applications are added to the desktop's menus or desktop shell via desktop entries, defined in text files with the .desktop extension (referred to as desktop files). The desktop environments construct menus for a user from the combined information extracted from available desktop entries.

Desktop files may be created in either of two places:

    `/usr/share/applications/` for desktop entries available to every user in the system
    `~/.local/share/applications/` for desktop entries available to a single user

You might need to restart GNOME for the new added applications to work.

Per convention, desktop files should not include spaces or international characters in their name.

Each desktop file is split into groups, each starting with the group header in square brackets ([]). Each section contains a number of key, value pairs, separated by an equal sign (=).

Below is a sample of desktop file:

```
[Desktop Entry]
Type=Application
Encoding=UTF-8
Name=Application Name
Comment=Application description
Icon=/path/to/icon.xpm
Exec=/path/to/application/executable
Terminal=false
Categories=Tags;Describing;Application
```

Explanation

    `[Desktop Entry]` the Desktop Entry group header identifies the file as a desktop entry
    `Type` the type of the entry, valid values are Application, Link and Directory
    `Encoding` the character encoding of the desktop file
    `Name` the application name visible in menus or launchers
    `Comment` a description of the application used in tooltips
    `Icon` the icon shown for the application in menus or launchers
    `Exec` the command that is used to start the application from a shell.
    `Terminal` whether the application should be run in a terminal, valid values are true or false
    `Categories` semi-colon (;) separated list of menu categories in which the entry should be shown

Command line arguments in the Exec key can be signified with the following variables:

    `%f` a single filename.
    `%F` multiple filenames.
    `%u` a single URL.
    `%U` multiple URLs.
    `%d` a single directory. Used in conjunction with %f to locate a file.
    `%D` multiple directories. Used in conjunction with %F to locate files.
    `%n` a single filename without a path.
    `%N` multiple filenames without paths.
    `%k` a URI or local filename of the location of the desktop file.
    `%v` the name of the Device entry.

Note that ~ or environmental variables like $HOME are not expanded within desktop files, so any executables referenced must either be in the $PATH or referenced via their absolute path.

A full Desktop Entry Specification is available at the GNOME Dev Center.

Launch Scripts

If the application to be launched requires certain steps to be done prior to be invoked, you can create a shell script which launches the application, and point the desktop entry to the shell script. Suppose that an application requires to be run from a certain current working directory. Create a launch script in a suitable to location (~/bin/ for instance). The script might look something like the following:
```
#!/bin/bash
pushd "/path/to/application/directory"
./application "$@"
popd
```

Set the executable bit for the script:

`$ chmod +x ~/bin/launch-application`

Then point the Exec key in the desktop entry to the launch script:

`Exec=/home/user/bin/launch-application`

