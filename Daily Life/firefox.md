

# Firefox

### Appearance

Firefox's user interface can be modified by editing the `userChrome.css` and `userContent.css` files in `~/.mozilla/firefox/<profile_dir>/chrome/` (*profile_dir* is of the form *hash.name*, where the *hash* is an 8 character, seemingly random string and the profile *name* is usually *default*). You can find out the exact name by **typing about:support**` in the URL bar, and searching for the `Profile Directory` field under the `Application Basics` section as described in the [Firefox documentation](https://support.mozilla.org/en-US/kb/profiles-where-firefox-stores-user-data#w_how-do-i-find-my-profile).

**Note:**

- The `chrome/` folder and `userChrome.css`/`userContent.css` files may not necessarily exist, so they may need to be created.
- **toolkit.legacyUserProfileCustomizations.stylesheets** must be enabled in **about:config**.

This section only deals with the `userChrome.css` file which modifies Firefox's user interface, and not web pages.

## Reference

* https://wiki.archlinux.org/title/Firefox/Tweaks