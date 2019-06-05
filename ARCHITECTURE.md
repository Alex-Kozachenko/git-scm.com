# git-scm.com architecture

This document describes the general setup and architecture that runs the
git-scm.com site. The idea is to document all the moving parts that
_aren't_ checked in to this repository. That may help new people joining
the project to help out, as well provide some continuity in case the
maintainer is hit by a bus.

## Content

Though the site is a rails app, it can _mostly_ be thought of a serving
static content. It's just that we suck in that static content and
pre-process it using nightly scheduled jobs. We never write anything to
the database on behalf of user requests.

The content is a mix of:

  - actual static content in this repository

  - community book content brought in from https://github.com/progit;
    see the `lib/tasks/book2.rake` file.

  - manpages from releases of the git project, imported and formatted
    via asciidoctor; see the `lib/tasks/index.rake` task.


## Heroku

The app itself is served by Heroku. The app name is `git-scm` (so you
can visit it directly as https://git-scm.herokuapp.com). The site is
owned by the git-scm.com team. If you want to be involved in managing
uptime/deploys/etc, you'll need a Heroku account and request to be added
to that team. The git-scm team receives credits from Heroku so that the
hosting is free.

We use a few Heroku add-ons:

  - Bonsai elasticsearch (see below)

  - Heroku Postgres as the database

  - Heroku Redis for rails caching

  - Heroku scheduler for cron jobs

The nightly scheduled jobs are:

  - `rake downloads` (pick up newly released git versions)

  - `rake preindex` (pull in and format manpages for released git
    versions)

  - `rake remote_genbook2` (pull in and format progit2 book content,
    including translations)

It should be safe to run any of those jobs more frequently. E.g., if you
know there's a new Git release out, then:

    heroku run rake preindex
    heroku run rake downloads

will get it on the site without waiting for the nightly run.

Merges to the `master` branch on GitHub auto-deploy to Heroku, so unless
you're doing something tricky you generally shouldn't need to manually
deploy.

Note that some of the formatting of manpages and book content happens
when they are imported by the rake tasks. So after fixing some
formatting and deploying, the rake jobs may need to be re-run with a
special flag to re-import (see the individual tasks for details).


## Cloudflare

We get enough requests that it's easy to overwhelm the single Heroku
dyno. So we have Cloudflare sitting in front of it, aggressively caching
everything. That also should make the site faster to serve to regions
far away from Heroku's servers.

The Cloudflare setup is mostly pretty simple:

 - they serve DNS for the whole domain (that's where they insert the CDN
   magic)

 - Cloudflare provides `https://` support to the user. Obviously the
   site is totally open and doesn't have any sensitive data, so this is
   really more about integrity. The certificate is generated by
   Cloudflare (and requires SNI on the browser side).

 - the Cloudflare connection to Heroku is passed over TLS; they provide an
   "internal" certificate that we ask Heroku to use, so the connection
   is secured between the two (again, mostly for integrity)

 - the most exotic config is that we use "page rules" to mark the whole
   site to be cached aggressively, regardless of any caching headers
   sent from Heroku. This is a bit of a hack, but there's very little on
   the site that can't be cached (which is perhaps a sign that the rails
   setup needs to be tweaked to send more reasonable caching headers,
   but this has been simple and effective so far).

   There are a few special page rules to lift this caching for cases
   where we do server-side logic (e.g.,
   https://github.com/git/git-scm.com/issues/1129#issuecomment-363067019"),
   but the long-term goal is to push that logic onto the client side as
   much as possible.

There's a single Cloudflare account/password that controls the site.
There's no team setup, but the information is in escrow with the Git PLC
at Software Freedom Conservancy. Cloudflare provides the project with
enough credits that it doesn't cost anything (though we're not using
very many features, so it's possible that a free account would be
sufficient, too).


## Bonsai Elasticsearch

The search functionality on the site is served by an elasticsearch
cluster. The index can be populated by running `rake search_index`
(manpages) and `rake search_index_book` (book) on Heroku (we only index
the manpages and book). This perhaps should be run nightly, or at least
after pulling in new content, but it currently isn't done automatically.

The elasticsearch cluster is provided by Bonsai via their Heroku plugin.
Our needs are larger than their free tier provides, but we receive
credits from them that provide the service for free.


## DNS

The actual DNS service is provided by Cloudflare (see above). The domain
itself is registered with Gandi, and is owned by the project via
Software Freedom Conservancy. Funds for the registration are provided
from the Git project's Conservancy funds, and both the Git PLC and
Conservancy have credentials to modify the setup.

Note that we own both git-scm.com and git-scm.org; the latter redirects
to the former.


## Manual Intervention

The site mostly just runs without intervention:

  - code merged to `master` is auto-deployed

  - new git versions are detected daily and manpages and download links
    updated

  - book updates (including translations) are picked up daily

There are a few tasks that still need to be handled by a human:

  - new images added to the book have to be copied manually from
    progit/progit2

  - new languages for book translations need to be added to
    `lib/tasks/book2.rake`

  - forced re-imports of content (e.g., a formatting fix to imported
    manpages) must be triggered manually