CKAN frontend in node.js.

[![Build Status](https://travis-ci.org/datopian/frontend-v2.svg?branch=master)](https://travis-ci.org/datopian/frontend-v2)
[![Coverage Status](https://coveralls.io/repos/github/datopian/frontend-v2/badge.svg?branch=master)](https://coveralls.io/github/datopian/frontend-v2?branch=master)

## Quick Start

Tested on NodeJS v8.10.0.

Clone the repo, install dependencies using yarn (or npm) and run the server:

```bash
yarn
yarn start
```

To run and watch HTML:

```bash
# note the -e which means we watch for changes in templates too
nodemon -e "js html" index.js
```

To run and watch everything:

```bash
npx gulp
```

This runs the server and watches for changes to CSS and icons. On changes to CSS [PostCSS](https://postcss.org/) and plugins ([postcss-import](https://github.com/postcss/postcss-import), [postcss-preset-env](https://preset-env.cssdb.org/), [tailwindcss](https://tailwindcss.com), [cssnano](https://cssnano.co/)) are run. On icon changes, the SVG icon sprite is remade. These processes can be run without watching via `npx gulp css` and `npx gulp icons`.

Icons are used like so: `<svg><use xlink:href="#<filename>" /></svg>`. Eg. to use the search.svg icon, you use `<svg><use xlink:href="#search" /></svg>`.

Note: Unless you are making significant changes to the design, you probably won't need these processes. Most styling is done through [Tailwind's](https://tailwindcss.com) utility classes.

## Set up your own backend

*By default, the app runs against mocked API so you don't need to setup your own backend.*

To change environment variables, you can rename `env.template` as `.env` and set the values. To unmock backends, you need to set `NODE_ENV` environment variable, e.g., `NODE_ENV=staging`.

### DMS

Setup `API_URL` environment variable so it points to your CKAN instance, e.g., for demo.ckan.org it would be:

```
export API_URL=https://demo.ckan.org/api/3/action/
```

### CMS

Use `WP_URL` environment variable to point to your WordPress instance. For example, we have test wordpress blog here https://edscms.home.blog/ so it would be:

```
export WP_URL=https://edscms.home.blog/
```

## API

*All of the controller and views use the API module - they don’t see backend.*

We have separated API module into `DmsModel` and `CmsModel`. The former part talks to CKAN (or can be any DMS), while the latter fetches content for static pages, for example, it can be WordPress. Below is the flow of how it works together:

```mermaid
sequenceDiagram
    Browser->>Controller: /
    Controller->>Browser: home.html

    alt Exists in CMS
      Browser->>Controller: /about
      Controller-->>CMS: slug=about
      CMS-->>Controller: found: page content
      Controller->>Browser: static.html

      Browser->>Controller: /news
      Controller-->>CMS: slug=news
      CMS-->>Controller: found: list of posts
      Controller->>Browser: blog.html

      Browser->>Controller: /news/my-blog-post
      Controller-->>CMS: slug=my-blog-post
      CMS-->>Controller: found: post content
      Controller->>Browser: static.html
    end
    alt Not Found in CMS
      Browser->>Controller: /search
      Controller-->>CMS: slug=search
      CMS-->>Controller: not found: 404
      Controller-->>DMS: search api
      DMS-->>Controller: result: list of data packages + summary
      Controller->>Browser: search.html

      Browser->>Controller: /org/gdp
      Controller-->>CMS: slug=org/gdp
      CMS-->>Controller: not found: 404
      Controller-->>DMS: getPackage api
      DMS-->>Controller: result: data package
      Controller->>Browser: showcase.html
    end
```

If above doesn't render, here is the screenshot:

![](https://i.imgur.com/TYdJaOS.png)

## Routes

Here is the summary of existing routes at the moment:

* Home: `/`
* Search: `/search`
  * with query: `/search?q=gdp`
* Showcase: `/organization/dataset`
* Organization: `/my_org`
  * It gets redirected from CKAN like path: `/organization/my_org`
* Collections: `/collections`
  * It gets redirected from CKAN groups page: `/group`
* CMS:
  * About: `/about`
  * Blog: `/news`
  * Post: `/news/my-post`
  * Anything else: `/foo/bar`

## Tests

Run tests (note that tests are running against mocked API_URL set to http://127.0.0.1:5000/api/3/action/):

```bash
yarn test

# watch mode:
yarn test:watch
```

## Deployment

*You can deploy this app to any host that supports NodeJS.*

### Zeit Now

Read the docs - https://zeit.co/examples/nodejs

Suggested config file (`now.json`):

```json
{
  "version": 2,
  "builds": [
    {
      "src": "index.js",
      "use": "@now/node-server",
      "config": { "maxLambdaSize": "50mb" }
    }
  ],
  "env": {
    "NODE_ENV": "development"
  },
  "routes": [
    {
      "src": "/(.*)",
      "dest": "/index.js"
    }
  ]
}
```

### Heroku

Read the docs about Deployment of NodeJS apps on Heroku - https://devcenter.heroku.com/articles/deploying-nodejs.

## Theming

By default, the frontend app uses DataHub theme which contains of templates (`/views/`) and assets (`/public/`).

If you need to customize the design of the site, you can create your own theme in the `/themes/` directory. E.g., we can create a theme called `example` with `public` and `views` directories so that they override default assets in `/public/` and templates in `/views/`:

```
/themes/example/public
/themes/example/views
```

To start using created template you need to set `THEME` environment variable to your theme name (you also can do it via `.env` file):

```
export THEME=example
```

---
**NOTE**

The default assets and templates are used, if a file isn't found in your theme. This allows you to change specific part of the templates or assets.

---

### Variables available in each page

#### Macros (helpers)

All default macros are located at `/views/_snippets.html`.

Importing macros in a template:

```html
{% import '_snippets.html' as snippets %}
```

Use it:

```html
{{ snippets.package_list_show(packages) }}
```

##### List data packages

Example on search page:

![](https://i.imgur.com/jECzGkG.png)

* Macros: `package_list_show`
* Parameters:
  * list of data packages
* Returns: list of HTML elements. Each element is sort of a summary card for a data package.

##### List data package licenses or sources

Example:

![](https://i.imgur.com/Wa3yMQc.png)


* Macros: `listify`
* Parameters:
  * list of standard data package licenses or sources
* Returns: HTML anchor tag or span element

#### Home page

Bespoke ...

#### Search page

```javascript
{
  title: 'Search',
  result: [list of data packages],
  query: {q: '', size: '', from: '', sort: ''},
  pages: [list of pages to display in pagination]
}
```

#### Showcase page

```javascript
{
  title: ...,
  dataset: datapackage, // a standard data package ...
  owner: {
    name: "rufuspollock"
    title: "Rufus Pollock",
    avatar: "...." // url to image for this
    route: "/rufuspollock",
  }
}
```

In standard Data package we have `created` and `modified` fields which is a timestamp and it isn't human readable (`2019-01-01 00:00:00`). We want to show it as `January 1, 2019`. The easiest would be to convert in controller and pass it through. E.g., in template if you access `dataset.created` it would print nicely formatted date.

#### Organization page

```javascript
{
  // org
  title: 'owner name',
  owner (name): 'owner name',
  description: 'description from profile',
  avatar: 'url to image',
  joinDate: 'eg, June 2019',

  // misc ...
  result: [list of data packages],
  query: {q: '', size: '', from: '', sort: ''},
  pages: [list of pages to display in pagination]
}
```

#### Collections page

List of collections page.

```javascript
{
  title: 'Dataset Collections',
  description: 'Catalogue of datasets ...',
  collections: [
    {
      name: '',
      title: '',
      summary: '',
      image: ''
    },
    ... // more collection objects
  ]
}
```

#### Individual collection page

```javascript
{
  title: 'title of collection',
  item: {
    name: '',
    title: '',
    summary: '',
    image: ''
  },

  // Misc
  result: [list of data packages],
  query: {q: '', size: '', from: '', sort: ''},
  pages: [list of pages to display in pagination]
}
```

#### Blog

List of posts.

```javascript
{
  posts: [{post}, ...]
}
```

``{post}`` object is a WP post object containing all available metadata. Below is main stuff that we use:

```javascript
{
  title: '',
  slug: '',
  content: '',
  date: ''
  modified: ''
}
```

#### Article / post page

```javascript
{
  title: '',
  content: '',
  published: 'formatted published date',
  modified: 'formatted modified date'
}
```

### i18n

#### Configure

Define location of translation files. We recommend creating `i18n` directory in your theme:

```
TRANSLATIONS=/themes/example/i18n
```

List of available locales then auto detected by filenames.

Use `defaultLocale` cookie to set the site's locale. E.g., if `defaultLocale=en`, then `/themes/example/i18n/en.json` file is used.

#### i18n of the site

In your templates:

```html
{{ __('Hello world!') }}
{{ __('Hi %s', 'you') }} // Hi you
{{ __('Hi {{ name }}', { name: 'you' }) }} // Hi you
```

This will add a phrase/word to your translation file if it is unknown.

Plurals translation:

```html
{{ __n('%s dog', 1) }} // 1 dog
{{ __n('%s dog', 3) }} // 3 dogs
```

In your translation file:

```json
{
  "%s dog": {
    "one": "%s dog",
    "other": "%s dogs"
  }
}
```

#### i18n of the content

You can have a page in WordPress with the same slug as original page plus locale at the end. E.g., for `/about` page we would have two pages on WP, `about` (English) and `about-da` (Danish). When fetching a content we can check the user's locale and get content in his/her language.
