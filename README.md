# Blog Articles

[![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/) 

*Backup of markdown files and images for [Elisa's blog](https://etsai.site).* 

## Setup

To install Node.js and hexo:

```shell
# install node.js
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
nvm install stable

# install hexo
npm install -g hexo-cli
echo 'PATH="$PATH:./node_modules/.bin"' >> ~/.profile
```

Initialize blog directory:

```shell
hexo init <blog name>
```

Install next theme:

```shell
git clone https://github.com/theme-next/hexo-theme-next themes/next
```

Change the information in two configure files.

Add `CNAME` under the `public` directory.

```
etsai.site
```

Then:

```shell
npm install hexo-deployer-git --save
```

## Personalize

### Images

Enable 

```xml
post_asset_folder: true
```

...in `_config.yml`.

### Indexing

Modify code in `node_modules/hexo-generator-index/lib/generator.js`, add:

```javascript
posts.data = posts.data.sort(function(a, b) {
    if(a.top && b.top) { 
        if(a.top == b.top) return b.date - a.date; 
        else return b.top - a.top;
    }
    else if(a.top && !b.top) { 
        return -1;
    }
    else if(!a.top && b.top) {
        return 1;
    }
    else return b.date - a.date; 
});
```

It should be something like:

```javascript
'use strict';

var pagination = require('hexo-pagination');

module.exports = function(locals) {
    var config = this.config;
    var posts = locals.posts.sort(config.index_generator.order_by);
    var paginationDir = config.pagination_dir || 'page';
    var path = config.index_generator.path || '';

    posts.data = posts.data.sort(function(a, b) {
        if (a.top && b.top) {
            if (a.top == b.top) return b.date - a.date; 
            else return b.top - a.top; 
        } else if (a.top && !b.top) { 
            return -1;
        } else if (!a.top && b.top) {
            return 1;
        } else return b.date - a.date; 
    });

    return pagination(path, posts, {
        perPage: config.index_generator.per_page,
        layout: ['index', 'archive'],
        format: paginationDir + '/%d/',
        data: {
            __index: true
        }
    });
};
```

Open `/blog/themes/next/layout/_macro/post.swig`, add:

```xml
          {% if post.top %}
            <i class="fa fa-thumb-tack"></i>
            <font color=A5A5A5>Pinned</font>
            <span class="post-meta-divider">|</span>
          {% endif %}
```

...under `<div class="post-meta">`.

Now we can use the `top` tag in the JSON header to pin certain articles.

### Add RSS Feed

```shell
npm install hexo-generator-feed --save
```



Configure this plugin in `_config.yml`.

```yaml
feed:
  type: atom
  path: atom.xml
  limit: 20
  hub:
  content:
  content_limit: 140
  content_limit_delim: ' '
  order_by: -date
  icon: icon.png
  autodiscovery: true
```

- **type** - Feed type. (atom/rss2)
- **path** - Feed path. (Default: atom.xml/rss2.xml)
- **limit** - Maximum number of posts in the feed (Use `0` or `false` to show all posts)
- **hub** - URL of the PubSubHubbub hubs (Leave it empty if you don't use it)
- **content** - (optional) set to 'true' to include the contents of the entire post in the feed.
- **content_limit** - (optional) Default length of post content used in summary. Only used, if **content** setting is false and no custom post description present.
- **content_limit_delim** - (optional) If **content_limit** is used to shorten post contents, only cut at the last occurrence of this delimiter before reaching the character limit. Not used by default.
- **order_by** - Feed order-by. (Default: -date)
- **icon** - (optional) Custom feed icon. Defaults to a gravatar of email specified in the main config.
- **autodiscovery** - Add feed [autodiscovery](http://www.rssboard.org/rss-autodiscovery). (Default: `true` )
  * Many themes already offer this feature, so you may also need to adjust the theme's config if you wish to disable it.

### Link format

Change `permalink` in `_config.yml` to:

```yaml
permalink: :title/
```

