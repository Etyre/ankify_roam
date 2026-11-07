# Ankify Roam

A command-line tool which brings flashcards created in [Roam Research](https://roamresearch.com/) to [Anki](https://apps.ankiweb.net/). Tag a block `#ankify` in Roam, export the graph, run `ankify_roam add <export>` and the block becomes an Anki note (front/back or cloze), updated in place on later runs. The upstream project was written by Taylor Mitchell (May 2020, [taylormitchell/ankify_roam](https://github.com/taylormitchell/ankify_roam)); this folder is Eli's fork ([Etyre/ankify_roam](https://github.com/Etyre/ankify_roam)), forked in August 2026 to fix cloze-hint parsing and then extended (only write the fields ankify_roam owns, write the parent block uid, skip auto-anki's state blocks, purple-flag unsuspended cards; see [CHANGELOG.md](CHANGELOG.md)). It is what the daily `com.elityre.roam-anki-autoimport` launchd job runs to keep Anki in sync with the Roam graph.

<img src="img/anki_roam_screenshot.png">

## Main Features

- Create front/back and cloze deletion flashcards in Roam and import to Anki.
- Supports block references, images, and aliases.
- [Include parent blocks](#Show-parent-blocks) on your Anki cards 
- Make edits in Roam to flashcards you've already imported and sync the changes to Anki. 
- Uses similar HTML syntax to Roam so you can style your Anki cards just like you do Roam.
- [Add color to or hide cloze deletion markup in Roam](#Add-color-or-hide-cloze-deletions-in-Roam).

## Contents
1. [Main Features](#Main-Features)
1. [How it works](#How-it-works)
    1. [Installation](#Installation)
    1. [Requirements](#Requirements)
    1. [Basic Usage](#Basic-Usage)
    1. [Options](#Options)
    1. [Customize Anki and Roam](#Customize-Anki-and-Roam)
    1. [Sync Automatically](#Sync-Automatically)
1. [Files](#Files)
1. [Runtime locations](#Runtime-locations)
1. [Common tasks](#Common-tasks)
1. [Settings that matter](#Settings-that-matter)
1. [Gotchas learned](#Gotchas-learned) (includes the old [Problems](#Problems) section)

## How it works

1. `RoamGraph.from_path` (`ankify_roam/roam/containers.py`) loads a Roam JSON export (or the zip holding it, or the newest export in a directory) into `Page` and `Block` objects. `ankify_roam/roam/content.py` parses each block's string into Roam markup objects (page refs, block refs, clozes, images, aliases, code, ...) that know how to render themselves as Anki-style HTML.
2. `RoamGraphAnkifier` (`ankify_roam/ankifiers.py`) first checks the connection and parameters: AnkiConnect must answer on `localhost:8765`, the deck must exist, and the two note types must exist and have a `uid` field. It then selects every block that carries the ankify tag itself (not inherited), minus blocks tagged with the dont-ankify tag and minus blocks tagged `#auto-anki-state`.
3. `BlockAnkifier` converts each selected block into an Anki note dict: a cloze note if the block contains cloze markup, otherwise a basic note with the block as the front and its children as the back. Per-block `#[[ankify_roam: key=value]]` tags override deck, note type, num-parents, pageref-cloze, download-imgs and suspend.
4. `ankify_roam/anki.py` talks to AnkiConnect. For each note it searches Anki for `uid:<block uid>`: if found and the owned fields differ it calls `updateNoteFields`, if found and identical it counts the note as unchanged, otherwise `addNote`. Images are uploaded with `storeMediaFile`. Suspend tags map to `suspend`/`unsuspend`, and unsuspended cards get the purple flag.
5. Every failure is logged with a traceback and counted; the run ends with one `Results: N notes added, N updated, N unchanged, N failed` line. The tool never deletes anything in Anki.

### Installation

```
pip install ankify_roam
```

To install this fork instead of the PyPI release, from this folder:

```
/usr/bin/python3 -m pip install --user .
```

### Requirements

- Python >=3.6
- [Anki](https://apps.ankiweb.net/)
- [AnkiConnect](https://ankiweb.net/shared/info/2055492159) (add-on for Anki)
- Python packages `bs4` (BeautifulSoup) and `requests`, installed by `pip install`. BeautifulSoup finds the `<img>` tags in a rendered field when `--download-imgs` is on; `requests` downloads them.

### Basic Usage

#### 1. Ankify Roam

Ankify a block (ie. flag it to go to Anki) by adding the #ankify tag to it. The tag must be included in the block itself, *it cannot be inherited from it's parents.*

By default, the block will be converted into a front/back style Anki note with the block content on the front and it's children on the back:

> - What is the capital of France? #ankify
>     - Paris

If the block includes any [cloze deletions](https://docs.ankiweb.net/#/editing?id=cloze-deletion), ankify_roam converts it to a cloze style Anki note. Add a cloze deletion by surrounding text in curly brackets: 

> {Paris} is the capital and most populous city of {France}, with a estimated population of {2,148,271} residents #ankify

In the example above, ankify_roam will add incremental cloze ids for each cloze deletion. But you can also explicitly define them (or a mixture of both). Here's an example showing what cloze markup in Roam becomes in Anki:


<table border=0px>
<tr>
<td>
        <div>{Paris} is the capital and most populous city of {2:France}, with a estimated population of {2,148,271} residents #ankify</div>
</td>
<td><div>→</div></td>
<td>
    <div>
        <div>{{c1::Paris}} is the capital and most populous city of {{c2::France}}, with a estimated population of {{c3::2,148,271}} residents #ankify</div>
    </div>
</td>
</tr>
</table>

Cloze ids matching the following patterns are all supported by ankify_roam: "c1:", "c1|", "1:" 


#### 2. Export Roam

Once you've tagged all the blocks to ankify, export your Roam: 
1. Click on the "more options" button in the top right corner of Roam.
2. Select Export All > JSON > Export All to export your Roam graph.
3. Unzip the downloaded file.

#### 3. Open Anki

Open Anki. Make sure you're on the profile you'd like to add the cards to and that you've installed the [AnkiConnect](https://ankiweb.net/shared/info/2055492159) add-on.

#### 4. Create Roam note types (first time only) 

Create 2 new note types in Anki: 'Roam Basic' and 'Roam Cloze'. These are the note types which your flashcards in Roam will be added as. 

Steps to create a 'Roam Basic' note type:
- Go to Tools > Manage Note Types and click on "Add"
- Select the "Add: Basic" option the click "OK"
- Name it "Roam Basic"
- With `Roam Basic` selected, click on "Fields..." and add a field called "uid" 
- With `Roam Basic` selected, Click on "Cards..." 
- Replace the css in "Styling" with the contents of [roam_basic.css](css/roam_basic.css)
- Click "Save"

Steps to create a 'Roam Cloze' note type:
- Go to Tools > Manage Note Types and click on "Add"
- Select the "Add: Cloze" option the click "OK"
- Name it "Roam Cloze"
- With `Roam Cloze` selected, click on "Fields..." and add a field called "uid" 
- With `Roam Cloze` selected, Click on "Cards..."
- Replace the css in "Styling" with the contents of [roam_cloze.css](css/roam_cloze.css)
- Click "Save"

(You can also create your own note types, and have ankify_roam populate those. For details, see [Create custom note types](#create-custom-note-types).)

#### 5. Add the Roam export to Anki

```
ankify_roam add my_roam.json
```
(Replace "my_roam.json" with the filename of the json within the zip you downloaded in [step 2](#2.-Export-Roam))

Your flashcards should now be in Anki! 

#### 6. Repeat

Whenever you create new flashcards in Roam or edit the existing ones, repeat these same steps to update Anki with the changes.  

### Options

#### Roam Export Path

The path to your exported Roam graph can refer to the json, the zip containing the json, or the directory which the zip is in. When a directory is given, ankify_roam will find and add the latest export in it. In my case, all 3 of these commands do the same thing:
```
ankify_roam add my_roam.json
ankify_roam add Roam-Export-1592525007321.zip
ankify_roam add ~/Downloads
```

#### Choose a different ankify tag

To use a tag other than #ankify to flag flashcards, pass the tag name to `--tag-ankify`: 
```
ankify_roam add --tag-ankify=flashcard my_roam.json
```  

... and if there are some blocks which include the #flashcard tag but you actually *don't* want ankify_roam to ankify it, add another tag (eg. #not-a-flashcard) and then tell ankify_roam by passing it to `--tag-dont-ankify`: 
```
ankify_roam add --tag-ankify=flashcard --tag-dont-ankify=not-a-flashcard my_roam.json
```  

#### Change the default deck and note types

To import your flashcards to different note types than the default 'Roam Basic' and 'Roam Cloze', pass the note type names to `--note-basic` and `--note-cloze` (see [Create custom note types](#Create-custom-note-types) for details):
```
ankify_roam add --note-basic="My Basic" --note-cloze="My Cloze" my_roam.json
``` 

To import your flashards to a different deck than "Default", pass the deck name to `--deck`:
```
ankify_roam add --deck="Biology" my_roam.json
```

You can also specify the deck and note type on a per-note basis using tags in Roam: 

> - 2+2={4} #[[ankify_roam: deck="Math"]] #[[ankify_roam: note="Cloze for math"]]

(When a deck or note type is specified using a tag on the block, those will take precedence over the deck and note type specified at the command line.)

To suspend or unsuspend a block's cards from Roam, tag it with `#[[ankify_roam: suspend=True]]` or `#[[ankify_roam: suspend=False]]`. Cards of a block tagged `suspend=False` are also given a purple flag in Anki, so you can find them with the search `flag:7`. (The flag is not removed when the tag is removed; clear it in Anki.)

#### Show parent blocks

To show the parents of your ankified block, pass a number of parents (or "all") to `--num-parents`. 

Here's an example where we specified that all parents should be included:

```
ankify_roam add --num-parents=all Geography.json
```

<img src="img/anki_roam_with_parents.png" width=600px>

Notice that "Geography" is shown differently from the rest of the parents. By default, the top level parent is shown as a title and all other parents are shown as breadcrumbs underneath. Because we included all parents, the top level parent for both blocks was the page name. But that's not always the case, as I'll show in the next example.

You can also use a tag to specify the `num-parents` on a single block. In this example, the `num-parents` was set to 2 using an inline tag:

<img src="img/anki_roam_n_parents.png" width=600px>

This ankified block has 3 parents: first parent is "[[Frace]]", the second is "Capitals", and the third is "Geography". Since `num-parents` was set to 2, only "[[Frace]]" and "Capitals" was included. In this case, "Capitals" was the top most parent included, so it's now the one displayed as a title. 


#### Cloze delete the base name only

When you add a cloze deletion around a namespaced page reference, eg. 

<img src="img/pageref_cloze_roam.png" width=600px>

... you can tell ankify_roam to only cloze delete the base name part of the page reference, leaving out the namespace, eg.

<img src="img/pageref_cloze_anki.png" width=500px>

... by setting the `--pageref-cloze` option to "base_only":
```
ankify_roam add --pageref-cloze=base_only my_roam.json
```

You can also set this on an individual note:

> - The {[[Design Pattern/Adaptor Pattern]]} specifies... #[[ankify_roam: pageref-cloze="base_only"]] 

#### Other options

`ankify_roam add --help` lists everything. Besides the options above: `--tag-ankify-root` (treat a block as the root even if it has parents, default `ankify-root`), `--include-page` (put the page title on the note), `--tags-from-attr` (assign tags listed next to a `tags::` attribute to the parent), `--max-depth` (how many levels of children to include on the back; `None` means all), and `--download-imgs` (`once`, `always` or `never`: whether embedded images are downloaded and stored in Anki's media folder, so cards work offline and on mobile).

### Customize Anki and Roam

#### Create custom note types

As mentioned in the [options](#Change-the-default-deck-and-note-types) section, you can import to different note types than the default 'Roam Basic' and 'Roam Cloze' types [provided](#4.-Create-Roam-note-types-(first-time-only)). Those note types will need to satisfy 2 requirements to be compatible with ankify_roam:   

1. **The first field(s) is for content from Roam (first 2 for Basic and 1 for Cloze)**. When ankify_roam converts a Roam block into an Anki note, it takes the content of the block and places it into the first field of the Anki note. For basic notes, it also takes the content of the block's children and adds them to the second field. The names of these fields doesn't matter, it just matters that they come first in the field order.

1. **Include an additional field called "uid"**. In addition to those fields, a "uid" field is required. This field is used by ankify_roam to remember which block in Roam corresponds with which note in Anki. Without this field, when you make a change to a block in Roam, ankify_roam will add that block as a new note in Anki rather than updating the existing one.

If the note type also has a field called **"parent uid"**, this fork fills it with the uid of the block's parent (or the page title for top-level blocks, since Roam pages have no uid in the export). Any other fields on the note type are neither read nor written, so you can keep your own notes in them.

If you are going to make your own note types, I'd suggest you create [clones](https://docs.ankiweb.net/#/editing?id=adding-a-note-type) of the 'Roam Basic' and 'Roam Cloze' note types and then just edit the style of those clones (see [here](https://www.youtube.com/watch?v=F1j1Zx0mXME&yt:cc=on) for a tutorial).


#### CSS ideas for your Anki cards

Hide all Roam tags (eg. the #ankify tag)
```css
.rm-page-ref-tag {
    display: none;
}
```

Hide page reference brackets.
```css
.rm-page-ref-brackets {
    display: none;
}
```

When a block has multiple children, they're added as bullet points on the backside of a card. If you'd prefer not to show the bullets, similar to the "View as Document" option in Roam, use the following CSS:
```css
.back-side ul {
    list-style-type: none;
    text-align: left;
    margin-left: 0;
    padding-left: 0;
}
```


#### Add color or hide cloze deletions in Roam

You can also define cloze deletions using curly bracket inside square brackets:

<img src="img/page_link_clozes.png" width=600px>

The nice thing about doing it this way is that you can now style the cloze markup. 

For example, you can make the cloze brackets only faintly visible by:
1. Pressing `Ctrl-C Ctrl-B` in Roam to hide the square brackets surrounding page links.
2. Adding this css to your [[roam/css]] page (how to [video here](https://www.youtube.com/watch?v=UY-sAC2eGyI)) to change the color of the curly brackets:
```css
span[data-link-title="{"] > span,
span[data-link-title="}"] > span
{
  color: #DDDCDC !important;
}
```

Now the block shown above will look like this: 

<img src="img/page_link_clozes_better.png" width=600px>

Note: Just like the regular cloze markup, the page links can also include cloze ids eg. [[{c1:]]Paris[[}]] 

[js/clozify.js](js/clozify.js) is a companion snippet for Roam (paste it into a `{{[[roam/js]]}}` block): with text selected in a block, Cmd-Shift-Z wraps it in `[[{N]]`...`[[}]]` with the next free cloze id and appends `#ankify` if the block doesn't have it yet.

### Sync Automatically 
It is possible to set up automatic updates of Anki using [Roam To Git](https://github.com/MatthieuBizien/roam-to-git). 

Follow the instructions on the Roam to Git page for setting up an automatically updating repository on GitHub. Clone that repository to your local machine: 
```git clone https://github.com/YOURNAME/notes```

Now you can run 

```ankify_roam add /PATH_TO_YOUR_REPO/notes/json/YOURDBNAME.json``` 

And further, you can add the git update to crontab:  
```echo "15 * * * * 'cd PATH_TO_YOUR_REPO;git pull;PATH_TO_ANKIFY/ankify_roam add PATH_TO_YOUR_REPO/json/YOURDBNAME.json '" | crontab ```  

Now you'll have git Roam to Git cloning your notes from Roam on the hour, and fifteen minutes later any updates/new items will be pulled in Anki, as long as it is running.

**How Eli's Mac does it.** A launchd agent, `com.elityre.roam-anki-autoimport`, runs `~/Library/Scripts/roam-anki-autoimport.sh` at 5:00, 9:00, 13:00 and 17:00 (if the Mac was asleep, launchd fires at the next wake). The script exits at once if a run already succeeded today (stamp file), otherwise it waits for the network, clones the Roam backup repo `Etyre/roam-backup-JSON-only` (`eli.json`, pushed by Roam To Git, ~107 MB via git-lfs), runs `ankify_roam add --deck="Daily review::Roam sync" roam-backup-JSON-only/eli.json` with the pip-installed copy of this package, writes the stamp, and asks AnkiConnect to sync to AnkiWeb. If Anki was not running it opens Anki first and quits it afterwards via AppleScript. The plist and script are not part of this repo (see [Runtime locations](#Runtime-locations)).

## Files

| File | Purpose |
|---|---|
| `ankify_roam/cli.py` | Entry point (`ankify_roam` console script). Parses the `add` and `init-models` sub-commands and their options. |
| `ankify_roam/ankifiers.py` | `RoamGraphAnkifier` (select blocks, add/update/suspend/flag notes, count results) and `BlockAnkifier` (turn one block into an Anki note dict, apply per-block option tags, download images). |
| `ankify_roam/anki.py` | Thin AnkiConnect client: `_invoke` plus one function per action (find/add/update notes, media, tags, models, decks, flags) and the exception classes. |
| `ankify_roam/default_models.py` | The built-in 'Roam Basic' and 'Roam Cloze' note types (fields, templates, CSS) and `add_default_models` used by `init-models`. |
| `ankify_roam/roam/containers.py` | `RoamGraph`, `Page`, `Block`: load an export (json, zip or directory), build the parent/child tree, query blocks, `parent_uid`. |
| `ankify_roam/roam/content.py` | Parser for Roam block strings: page refs, block refs, clozes, aliases, images, code, checkboxes, attributes, ... each rendered to HTML. |
| `ankify_roam/util.py` | `get_default_args` (reads a function's defaults so the CLI defaults match the class defaults). |
| `ankify_roam/_version.py` | Version string; the CLI appends the git branch when run from a checkout. |
| `tests/unit/` | 88 unit tests for the parser, containers and ankifiers; no Anki needed. |
| `tests/integration/test_anki.py` | End-to-end tests that open Anki on a profile named `test` and add/remove real notes. Needs `psutil` and a `test` profile. |
| `tests/export-pages.json` | Small Roam export used by the tests and the VS Code "Test graph" launch config. |
| `css/roam_basic.css`, `css/roam_cloze.css` | Styling for the two note types, for pasting into Anki. Generated from `default_models.py` by `create_css.py`. |
| `create_css.py` | Regenerates the two css files from `default_models.py`. |
| `js/clozify.js` | Roam `roam/js` snippet: Cmd-Shift-Z wraps the selection in cloze markup. |
| `setup.py` | Package metadata and the `ankify_roam` console script. |
| `CHANGELOG.md` | Release notes, including this fork's changes under "Unreleased" and 0.2.3. |
| `.github/workflows/` | Upstream CI: run unit tests on push, publish to PyPI and create a GitHub release on tag. |
| `.vscode/` | Upstream's launch configs (`add` against a real graph / the test graph) and unittest settings. |
| `img/` | Screenshots used in this README. |

## Runtime locations

Everything below is on Eli's Mac; the package itself has no config file or state of its own.

| What | Where |
|---|---|
| This checkout (the fork) | `~/Documents/agent-coding/ankify_roam` |
| Installed copy the daily job runs | `/Users/elityre/Library/Python/3.9/bin/ankify_roam`, package in `/Users/elityre/Library/Python/3.9/lib/python/site-packages/ankify_roam/` (installed with `/usr/bin/python3 -m pip install --user .`; identical to this checkout as of 2026-11-07) |
| launchd plist | `~/Library/LaunchAgents/com.elityre.roam-anki-autoimport.plist` |
| Job script | `~/Library/Scripts/roam-anki-autoimport.sh`, a copy of `~/Documents/Programming/Incremental_Reading_in_Roam/bash_scripts/current_autoimport_script-checks_if_anki_is_open.sh`. Re-copy after editing the original (launchd cannot read `~/Documents`). |
| Job log | `~/Library/Logs/roam-anki-autoimport.log` (stdout and stderr; ankify_roam's own lines are timestamped) |
| Last-success stamp | `~/Library/Logs/roam-anki-autoimport.last-success` (today's date once a run has succeeded) |
| Temporary clone | `~/roam-backup-JSON-only/` during a run, deleted afterwards |
| Anki side | AnkiConnect on `http://localhost:8765`; deck `Daily review::Roam sync`; note types `Roam Basic` and `Roam Cloze`; downloaded images in Anki's `collection.media` |

## Common tasks

```bash
# Run the unit tests (no Anki needed; /usr/bin/python3 has bs4 and requests)
cd ~/Documents/agent-coding/ankify_roam && /usr/bin/python3 -m unittest discover tests/unit

# Run this checkout (not the installed copy) against the small test graph.
# Needs Anki open with the deck and note types present; adds real notes to the given deck.
cd ~/Documents/agent-coding/ankify_roam && /usr/bin/python3 -m ankify_roam.cli add --deck="Default" tests/export-pages.json

# Run the installed copy against a Roam export
ankify_roam add --deck="Daily review::Roam sync" ~/Downloads        # newest export in the folder
ankify_roam add --help

# Create the default note types in the current Anki profile
ankify_roam init-models            # add --overwrite to replace existing ones

# Update the installed copy after changing code in this checkout
cd ~/Documents/agent-coding/ankify_roam && /usr/bin/python3 -m pip install --user .

# Daily job: tail the log, see when it last succeeded, force a rerun today
tail -f ~/Library/Logs/roam-anki-autoimport.log
cat ~/Library/Logs/roam-anki-autoimport.last-success
rm ~/Library/Logs/roam-anki-autoimport.last-success && launchctl kickstart gui/$(id -u)/com.elityre.roam-anki-autoimport

# Daily job: load / unload the agent after editing the plist
launchctl bootout gui/$(id -u)/com.elityre.roam-anki-autoimport
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.elityre.roam-anki-autoimport.plist

# Regenerate css/ from default_models.py
cd ~/Documents/agent-coding/ankify_roam && /usr/bin/python3 create_css.py

# Integration tests (opens and kills Anki, uses a profile named "test"; do not run while Anki is in use)
/usr/bin/python3 -m unittest discover tests/integration
```

Test path: the unit tests are the no-Anki proof that parsing and note generation work. The end-to-end check is the `add` command against `tests/export-pages.json` above, or the integration tests on a throwaway `test` profile.

## Settings that matter

- **Anki**: the [AnkiConnect](https://ankiweb.net/shared/info/2055492159) add-on must be installed and Anki open on the right profile; nothing works without `localhost:8765` answering. The target deck and the two note types must already exist, and each note type needs a `uid` field (`init-models` creates the default ones). The daily job targets deck `Daily review::Roam sync`.
- **Roam**: blocks need the `#ankify` tag on the block itself. Roam To Git pushes the graph to `Etyre/roam-backup-JSON-only` on GitHub, which is what the daily job imports; if that stops updating, Anki silently stops getting new cards.
- **macOS / launchd**: the job script must live outside `~/Documents` (TCC blocks launchd from reading it), `git-lfs` must be installed (`~/.local/bin`, on the plist's PATH) or the clone yields a 134-byte pointer file instead of `eli.json`, and the job cannot send keystrokes, so AnkiWeb sync is triggered through AnkiConnect's `sync` action instead of osascript.
- **Python**: the installed copy lives in the system Python 3.9 user site (`/usr/bin/python3`). Homebrew's python3 does not have `bs4`/`requests`, so use `/usr/bin/python3` for tests and installs.

## Gotchas learned

### Missing Features
- No LaTeX support
- No syntax highlighting for code blocks

### Non-Intuitive Behaviour
- If you change a flashcard's field content in Anki, that change will be overwritten with whatever is in Roam the next time you run ankify_roam. So make those changes in Roam, not Anki.
- When a flashcard in Roam has already been imported to Anki, the only changes made in Roam which will be reflected in Anki are changes to the fields. Changes to it's tags, deck, and note type need to be done manually in Anki.
- If you move the content of a block into a new block in Roam, ankify_roam will treat that as a new flashcard. This is because ankify_roam uses the block uid and the Anki uid field to know which block corresponds with which Anki note.
- Deleting a flashcard in Roam doesn't delete it in Anki. You'll need to delete it in Anki manually.
- A flashcard deleted in Anki will be re-imported to Anki next time you run ankify_roam if you don't also delete it or remove the #ankify tag in Roam. 
- When you let ankify_roam infer the cloze ids, you can get some weird behaviour when you add a new cloze deletion to a note in Roam which was already imported to Anki. For example, if you have "Paris is the capital of {France}" in Roam, that'll become "Paris is the capital of {{c1::France}}" in Anki. Later, if you add a cloze deletion around Paris ie. "{Paris} is the capital of {France}", ankify_roam will convert that into "{{c1::Paris}} is the capital of {{c2::France}}". Notice that the "France" cloze id is now "c2" instead of "c1". This is because ankify_roam assigns cloze ids in the order that the cloze deletions appear. The result is that in Anki the original flashcard will now cloze delete "Paris" instead of "France" and a new flashcard will be added which cloze deletes "France". To avoid this, explicitely add cloze ids in Roam which match the existing note in Anki eg. "{2:Paris} is the capital of {1:France}"

### Learned on Eli's setup
- **"N failed" in the daily log is mostly `DuplicateError: cannot create note because it is a duplicate`.** It happens when a block's uid is not found in Anki but a note with the same first field already exists (the block was re-created in Roam, or the Anki note lost/changed its uid). Anki refuses the add; the old note stays. Fix by hand: delete the stale note in Anki or fix its `uid` field. The same handful of blocks fail every day until that is done. Rarer: `cannot create note for unknown reason` and `One or more notes failed validation`; cause not investigated, and the same blocks later show up as duplicates once a note exists.
- **Earlier versions blanked user fields.** Before 0.2.3 every field the note type had was sent on update, so anything typed into an extra field (e.g. `Extra`) in Anki was wiped on the next sync. Now only Front/Back or Text, `uid` and `parent uid` are written.
- **auto-anki's state block became cloze notes.** auto-anki (another tool) stores the text of every note it wrote, `#ankify` included, in a hidden block tagged `#auto-anki-state`; ankify_roam turned each one into a bogus cloze note. Blocks with that tag are now skipped (`TAGS_NEVER_ANKIFY` in `ankifiers.py`).
- **Cloze hints written `{text[[::]]hint}`** (what Roam produces when `[[` auto-closes) used to leak `[[` into the answer; fixed in this fork.
- **The installed copy is not this checkout.** The daily job runs the pip-installed package in `~/Library/Python/3.9`. Editing code here does nothing for the job until you run `/usr/bin/python3 -m pip install --user .`. `pip install ankify_roam` from PyPI would replace it with upstream 0.2.2 and lose the fork's fixes.
- **Modern Anki is a python process**, so `pgrep -x anki` no longer finds it; the job script also checks `pgrep -f AnkiProgramFiles`. Likewise `pkill -x anki` no longer quits it; the script uses `osascript -e 'tell application "Anki" to quit'`. If Anki is open on a profile other than the one with the deck, the run fails with `Deck named ... not in Anki`.
- **`User canceled. (-128)` in the log** is the AppleScript quit being declined (Anki showed a dialog). Harmless; the import had already succeeded.
- **launchd and `~/Documents`**: a launchd job cannot read scripts under `~/Documents` (TCC), which is why the script is copied to `~/Library/Scripts`. Edit the original and re-copy, or the two drift.
- **git-lfs**: without `git-lfs` on PATH the clone succeeds but `eli.json` is a pointer file; the script checks the file is > 1 MB and retries. The ~107 MB download sometimes fails with "Connection reset by peer" at first wake, hence the retry loop and network wait.
- **Version string**: `ankify_roam --version` prints `0.2.3+master` when run from a git checkout (branch appended) and plain `0.2.3` from the installed copy. `/usr/bin/python3` prints a urllib3 `NotOpenSSLWarning` on every run; it is noise.
