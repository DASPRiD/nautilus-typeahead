# nautilus-typeahead

A rewrite of the typeahead patch the AUR `nautilus-typeahead` package carries,
against Nautilus 51.1.

Type a few letters in a folder and it jumps to the first file whose name starts
with them. A small entry shows up bottom left with what you typed, and you can
edit it.

Not aimed at upstream; GNOME has turned typeahead down before. The point is a
patch that's small enough to re-apply every release without becoming a puzzle.

`patches/0002-Type-ahead-find.patch` is the new one. `patches/0001` is the
existing backspace patch, untouched and byte-identical down to the `b2sum`
already in the PKGBUILD, and only here so the PKGBUILD works as given. The
PKGBUILD diff is four lines: a contributor line, the `0002` filename, its
checksum, and dropping the `gawk -i inplace` schema rewrite, which isn't needed
any more.

## What it fixes

The old patch reused `NautilusQueryEditor` as the typeahead box but never
revealed it, so keys went into a `GtkText` that wasn't in the widget tree.
Nothing to see, nothing to edit. And with nothing holding the focus, backspace
fell through to `slot.up` and navigated up instead of deleting a character.

This uses its own `GtkSearchEntry` and gives it the focus on the first
keystroke, which is what makes backspace work: a focused `GtkText` takes the key
before the window gets to managed shortcuts, same as in the Ctrl+L location bar.
The backspace patch needed no changes.

Matching was wrong for non-ASCII names. The old code called
`g_utf8_casefold (name, text_len)`, passing the query's length in *bytes* as the
length of the name, so a one-byte query against `Äpfel` casefolded half a
character, and nothing was normalized.

It also looked in the wrong place. The old patch read
`nautilus_directory_get_file_list()`, the current folder, then sorted the matches
itself using a sort state it had to add `NautilusFilesView` API to read. That
misses expanded folders in list view and can land on a hidden file that isn't on
screen. This walks `NautilusViewModel`, which is already sorted, already
filtered, and already has expanded folders flattened in place, so sort order,
subfolders and hidden files all come for free and
`nautilus_files_view_get_sort_state()` is gone.

## The setting

`org.gnome.nautilus.preferences type-ahead-find`, default true, switch in
Preferences.

Deliberately not the old `type-ahead-search`, which meant the opposite: there
`true` meant "typing starts a recursive search" and the PKGBUILD rewrote the
default to `false` to switch typeahead on. Keeping that name would silently flip
behavior for everyone upgrading, since dconf values survive.

Off, no key capture widget is installed and typing starts a search exactly like
an unpatched build.

## Keys

Up/Down and Ctrl+G / Ctrl+Shift+G walk the matches, Enter opens, Escape closes
and keeps the selection. Backspacing to an empty query closes it too, otherwise
the entry keeps the focus and the arrow keys can't reach the list. It gives up
after five seconds of nothing, which is what the GTK 3 tree view used.

## Notes for rebasing

All the logic is in `src/nautilus-typeahead.c`. Of 556 added lines, 505 are that
file and its header; existing files take the other 51, spread over 8 of them,
and nothing is deleted or rewritten.

`nautilus_files_view_get_private_model()` and
`nautilus_files_view_get_private_list_base()` are marked `/* testing-only */` in
`nautilus-files-view.h` and are the two accessors the entry needs, so a rename
breaks the build loudly. `find_match_position()` is the only thing that cares
how `NautilusViewModel` is layered.

GTK 4.24 is a hard floor: `GtkEditable::input-intercepted` is new in it, and
`gtk_search_entry_set_key_capture_widget()` only hands the entry's `GtkIMContext`
to the capture widget from 4.24 on, which is what makes dead keys work before
the entry has the focus. A GTK release that reworks that could leave this
compiling but misbehaving.

`nautilus/` holds the patch as one commit on the `typeahead` branch.
`git format-patch --no-signature --start-number 2 -o ../patches 51.1..typeahead`
regenerates `0002`, then `updpkgsums`.

## Known limitations

While the entry is open the arrow keys belong to it: Up/Down walk the matches,
Left/Right move the text cursor. Stepping through items one at a time isn't
available until Escape, which shows up most in grid view, where Left/Right are
the usual way to move. GTK 3's tree view search bound Up/Down the same way.

Space can't start a query, because GTK's interceptor skips it. Once the entry is
open, space types normally.

If nothing starts with what you typed it falls back to a substring match, so
`report` finds `2024-report.pdf`. Prefix always wins, so it only applies where
there'd be no match otherwise.

With the search bar open, clicking a result puts the focus in the view and
typing then runs typeahead over the results, where stock Nautilus appends to the
search query. Left that way on purpose: finding a name in a long result list is
useful, and Ctrl+F leaves the focus in the search entry anyway.

Every displayed name is folded on every keystroke. Worst case is 3 ms for 10 000
files, but a folder ten times that would want them cached on the view item.
