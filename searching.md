# Keyboard Shortcuts

## Searching
One of the primary purposes of Dashy is to allow you to quickly find and launch a given app. To make this as quick as possible, there is no need to touch the mouse, or press a certain key to begin searching - just start typing. Results will be filtered in real-time. No need to worry about case, special characters or small typos, these are taken care of, and your results should appear.

## Navigating
You can navigate through your items or search results using the keyboard. You can use <kbd>Tab</kbd> to cycle through results, and <kbd>Shift</kbd> + <kbd>Tab</kbd> to go backwards. Or use the arrow keys, <kbd>↑</kbd>, <kbd>→</kbd>, <kbd>↓</kbd> and <kbd>←</kbd>.

## Launching Apps
You can launch a elected app by hitting <kbd>Enter</kbd>. This will open the app using your default opening method, specified in `target` (either `newtab`, `sametab`, `modal` or `workspace`). You can also use <kbd>Alt</kbd> + <kbd>Enter</kbd> to open the app in a pop-up modal, or <kbd>Ctrl</kbd> + <kbd>Enter</kbd> to open it in a new tab. For all available opening methods, just right-click on an item, to bring up the context menu.

## Tags
By default, items are filtered by the `title` attribute, as well as the hostname (extracted from `url`), the `provider` and `description`. If you need to find results based on text which isn't included in these attributes, then you can add `tags` to a given item. 

```yaml
  items:
  - title: Plex
    description: Media library
    icon: favicon
    url: https://plex.lab.local
    tags: [ movies, videos, music ]
  - title: FreshRSS
    description: RSS Reader
    icon: favicon
    url: https://freshrss.lab.local
    tags: [ news, updates, blogs ]

```

In the above example, Plex will be visible when searching for 'movies', and FreshRSS with 'news'


## Custom Hotkeys
For apps that you use regularly, you can set a custom keybinding. Use the `hotkey` parameter on a certain item to specify a numeric key, between `0 - 9`. You can then launch that app, by just pressing that key, which is much quicker than searching for it, if it's an app you use frequently.

```yaml
- title: Bookstack
  icon: far fa-books
  url: https://bookstack.lab.local/
  hotkey: 2
- title: Git Tea
  icon: fab fa-git
  url: https://git.lab.local/
  target: workspace
  hotkey: 3
```

In the above example, pressing <kbd>2</kbd> will launch Bookstack. Or hitting <kbd>3</kbd> will open Git in the workspace view.


## Clearing Search
You can clear your search term at any time, by pressing <kbd>Esc</kbd>.
This can also be used to close an open pop-up modal.
