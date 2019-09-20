# Blog Articles

[![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/) 

*Backup of markdown files and images for [Elisa's blog](https://etsai.site).* 

## Setup

To install Node.js and hexo:

```shell
./hexo_install.sh
# or

# install node.js
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
nvm install stable

# install hexo
npm install -g hexo-cli
echo 'PATH="$PATH:./node_modules/.bin"' >> ~/.profile
npm install hexo-deployer-git --save
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

```
          {% if post.top %}
            <i class="fa fa-thumb-tack"></i>
            <font color=A5A5A5>Pinned</font>
            <span class="post-meta-divider">|</span>
          {% endif %}
```

...under `<div class="post-meta">`.

Now we can use the `top` tag in the JSON header to pin certain articles.

