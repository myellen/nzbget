# Try the touch-friendly nzbget web UI on the Raspberry Pi

Instructions for a Claude Code session running on Max's Raspberry Pi.
Paste this whole file into Claude Code on the Pi.

## What you are doing

Max has a fork of nzbget with a small change to the web interface: on touch
screens wider than the phone layout (tablets, the inner screen of the iPhone
Duo), buttons, checkmarks, menus and table headers get larger tap areas, and
form fields use 16px text so iOS Safari stops zooming in on them. Mouse users
and the existing phone layout are unchanged.

- Fork: https://github.com/myellen/nzbget
- Branch: `feature/touch-friendly-webui`
- The change touches only three static files in the web interface:
  `webui/style.css`, `webui/light-theme.css`, `webui/dark-theme.css`.
  There is nothing to compile.

Your job is to let Max try it on the nzbget that already runs on this Pi,
in a way that is easy to undo, then hand him a short checklist to tap through
on his iPhone Duo.

## Ground rules

- Back up before you change anything, and tell Max where the backup is.
- Ask Max before each of these, and wait for his yes:
  - installing any package or tool,
  - editing nzbget's config file,
  - copying files into a running Docker container,
  - restarting nzbget, its service or its container,
  - deleting anything.
- Read-only checks (finding the install, reading versions and configs,
  downloading the patch to a scratch folder) need no question.
- Never print or copy nzbget's `ControlPassword` or any other password from
  the config. If you need to call the API, ask Max to run that command
  himself or to give you the credentials for this session.
- If anything does not match what this file expects, stop and tell Max what
  you found rather than improvising a bigger change.

## Step 1: Find the existing install

Work out, and tell Max in a few lines:

1. CPU architecture: `uname -m` (expect `aarch64` or `armv7l`).
2. How nzbget runs. Check in this order:
   - Docker: `docker ps --format '{{.Names}}\t{{.Image}}\t{{.Ports}}'`
     and look for an image named like `nzbgetcom/nzbget` or
     `linuxserver/nzbget`.
   - systemd: `systemctl list-units --all | grep -i nzbget`, then
     `systemctl cat <unit>` to see the binary and config path.
   - A plain process: `pgrep -a nzbget`.
3. The nzbget version: `nzbget -v` for a native install, or
   `docker exec <container> /app/nzbget/nzbget -v` for Docker (if that path
   does not exist, find the binary with
   `docker exec <container> sh -c 'command -v nzbget || find / -name nzbget -type f 2>/dev/null'`).
4. The config file in use (from the `-c` argument, the systemd unit, or
   `/config/nzbget.conf` inside Docker), and its `WebDir=` and `ControlPort=`
   lines. Read only those two lines; don't print the rest of the file.
5. The web interface folder actually in use: the `WebDir` value, or for
   Docker the folder that holds `index.html` and `style.css`, usually
   `/app/nzbget/webui`
   (`docker exec <container> sh -c 'find / -name index.html -path "*webui*" 2>/dev/null'`).

## Step 2: Get the change as a patch

Download the branch's diff into a scratch folder. This changes nothing on
the system:

```
mkdir -p ~/nzbget-duo && cd ~/nzbget-duo
curl -fsSL -o touch.diff \
  "https://github.com/myellen/nzbget/compare/develop...feature/touch-friendly-webui.diff"
grep '^+++ ' touch.diff
```

The last command should list only the three webui files named above. If it
lists anything else, stop and tell Max.

## Step 3: Make a patched copy of the Pi's own web interface

Patch a copy of the web interface that this Pi already serves, rather than
pulling the fork's whole `webui` folder, so the files keep matching the
nzbget version installed here.

Native install:

```
cp -a "<webui folder from step 1>" ~/nzbget-duo/webui-original
cp -a ~/nzbget-duo/webui-original ~/nzbget-duo/webui
cd ~/nzbget-duo && patch -p1 --dry-run < touch.diff
```

Docker:

```
docker cp <container>:<webui folder from step 1> ~/nzbget-duo/webui-original
cp -a ~/nzbget-duo/webui-original ~/nzbget-duo/webui
cd ~/nzbget-duo && patch -p1 --dry-run < touch.diff
```

`patch -p1` run from `~/nzbget-duo` applies `webui/...` paths to the copy in
`~/nzbget-duo/webui`. If the dry run reports no failed hunks, run the same
command without `--dry-run`. If hunks fail, the installed nzbget is too far
from the branch: stop and tell Max, and offer the fallback at the end of this
file.

`~/nzbget-duo/webui-original` is the backup of what is served today.

## Step 4: Serve the patched copy (ask Max first)

Pick the route that matches the install, explain it to Max in two or three
lines, and wait for his yes.

Native install (config change plus restart):

1. Back up the config: `cp -p <config> <config>.before-duo`.
2. Set `WebDir=` in the config to the absolute path of `~/nzbget-duo/webui`.
   Change only that line.
3. Make sure the user nzbget runs as can read that folder
   (`sudo -u <nzbget user> ls <path>`); if it can't, tell Max instead of
   changing permissions on his home folder.
4. Restart nzbget (`sudo systemctl restart <unit>`, or however step 1 showed
   it is run).

Docker (no config change):

1. Copy the three patched files into the container, over the originals:
   ```
   for f in style.css light-theme.css dark-theme.css; do
     docker cp ~/nzbget-duo/webui/$f <container>:<webui folder>/$f
   done
   ```
2. No restart should be needed, since nzbget reads the web files from disk.
   If the browser still shows the old look after a reload, ask Max before
   restarting the container. Note: recreating the container (for example
   `docker compose pull && up -d`) brings back the original files by itself.

## Step 5: Check it is being served

```
curl -s http://localhost:<ControlPort>/style.css | grep -c "TOUCH SCREENS"
```

This should print `2`. The web server may ask for a login on this URL; if it
does, ask Max to run the command himself rather than reading the password from
the config.

Then tell Max the address to open on the Duo
(`http://<Pi's LAN IP>:<ControlPort>/`, from `hostname -I`) and to open it in
a private tab, or reload twice, so Safari does not show cached styles.

## Step 6: Give Max this checklist

Send it to him as is. The outer screen uses nzbget's phone layout, which the
change leaves alone, so most checks are on the inner screen.

On the inner screen, in portrait and in landscape:

1. Tap a row's checkbox, and just around it. The row is selected, and no
   dialog opens.
2. Drag a row by its checkbox column to reorder the queue (if you use that),
   and check it still moves.
3. Tap a download's name. The edit dialog opens.
4. Tap anywhere in a Priority, Status or Category cell, not just on the text.
   Its menu opens.
5. Tap column headers (Name, Size, Age). The table sorts.
6. Tap the toolbar buttons (Add, Edit, Pause, Resume, Delete and the arrows).
   They are easy to hit and their labels are not cut off.
7. Tap the "records per page" box. The number is readable and the list
   opens.
8. Tap the search field at the top. Safari does not zoom in.
9. Open a download's edit dialog and tap a text field. Safari does not zoom in.
10. Settings: tap through the section list on the left, and tap a text field
    in a section. No zoom, and the right section opens.
11. Tap the small arrow next to the round play button. Its menu opens. Then
    tap the play button itself, including near its right edge. It pauses and
    resumes.
12. Tap the small arrow inside the search field. The filter menu opens.
13. History: rows with no category show "None" in grey, and tapping it opens
    the category menu.

On the outer screen and in Split View:

14. Everything looks and behaves as before.

Folding and unfolding:

15. Fold and unfold while the page is open. The layout switches cleanly
    between the phone and full layouts.

On a computer with a mouse:

16. Nothing looks different from before.

Ask Max to note anything that fails, with the screen, orientation and step
number.

## Undo

Native install:

```
cp -p <config>.before-duo <config>
sudo systemctl restart <unit>
```

Docker:

```
for f in style.css light-theme.css dark-theme.css; do
  docker cp ~/nzbget-duo/webui-original/$f <container>:<webui folder>/$f
done
```

Ask Max before restoring the config, copying into the container or
restarting. Leave `~/nzbget-duo` in place unless he asks you to delete it.

## Fallback: a separate test instance

Only if the patch does not apply to the installed version, and only with
Max's yes, since it installs or pulls software: run a second, throwaway
nzbget next to the real one, on port 6790, with its own config and download
folder, serving the fork's `webui` folder. With Docker, that is the
`nzbgetcom/nzbget:testing` image with
`-v ~/nzbget-duo/test-config:/config -v ~/nzbget-duo/test-downloads:/downloads -p 6790:6789`,
`WebDir` in that test config pointing at a bind mount of the fork's `webui`
folder, and `MainDir` inside `/downloads`. Without Docker, build the fork's
branch as `docs/LINUX.md` in the fork describes. Its queue starts empty, so
add one or two small test NZBs to have rows to tap. Stop and remove the test
instance when Max is done, again with his yes.
