---
title: Development Guide
icon: git-merge
summary: 'How to install Plume on your computer and make changes to the source code.
This guide also gives you tips for making debugging and testing easier.'
priority: 2
---

## Installing the development environment

Please refer to the [installation guide](/installation). Choose to compile Plume
from source when asked. Instead of using `cargo install`, use `cargo run` which
starts a freshly compiled debugging version of Plume.

It is recommended to enable the `debug-mailer` feature, especially if you need
emails during development. You can do it by passing the `--feature debug-mailer`
flags to `cargo`. When enabled, mails will be logged to the standard output instead
of being sent for real.

## Testing the federation

To test the federation, you'll need to setup another database,
also owned by the "plume" user, but with a different name. Then, you'll need to setup
this instance too.

The easiest way to do it is probably to install `plm` and `plume` globally (as explained
[here](/installation/with/source-code)), but with the `--debug` flag to avoid long compilation
times. Then create a copy of your `.env` file in another directory, and change the `DATABASE_URL`
and `ROCKET_PORT` variables. Then copy the migration files in this new directory and run them.

```
diesel migration run
```

Setup the new instance with `plm` [as explained here](/installation/config).

Now, all you need for your two instances to be able to communicate is a fake domain
name with HTTPS for each of them. The first step to have that on your local machine is
to edit your `/etc/hosts` file, to create two new aliases by adding the following lines.

```
127.0.0.1       plume.one
127.0.0.1       plume.two
```

Now, we need to create SSL certificates for each of these domains. We will use `mkcert`
for this purpose. Here are [the instructions to install it](https://github.com/FiloSottile/mkcert#installation).
Once you installed it, run.

```bash
mkcert -install
mkcert plume.one plume.two
```

Finally, we need a reverse proxy to load these certificates and redirect to the correct Plume instance for each domain.
We will use Caddy here as it is really simple to configure, but if you are more at ease with something else you can also
use alternatives.

To install Caddy, please refer to [their website](https://caddyserver.com/download). Then create
a file called `Caddyfile` in the same directory you ran `mkcert` and write this inside.

```
plume.one:443 {
  bind 127.0.0.1
  proxy / 127.0.0.1:7878 {
    transparent
  }
  tls plume.one+1.pem plume.one+1-key.pem
}

plume.two:443 {
  bind 127.0.0.1
  proxy / 127.0.0.1:8787 {
    transparent
  }
  tls plume.one+1.pem plume.one+1-key.pem
}
```

Eventually replace the ports in the `proxy` blocks by the one of your two instances, and
then run `caddy`. You can now open your browser and load `https://plume.one` and `https://plume.two`.

# Running tests

To run tests of `plume-models` use `RUST_TEST_THREADS=1`, otherwise tests are run
concurrently, which causes error because they all use the same database.

# Internationalization

To mark a string as translatable wrap it in the `i18n!` macro. The first argument
should be the catalog to load translations from (usually `ctx.1` in templates), the
second the string to translate. You can specify format arguments after a `;`.

If your string vary depending on the number of elements, provide the plural version
as the third arguments, and the number of element as the first format argument.

You can find example uses of this macro [here](https://github.com/Plume-org/gettext-macros#example)

# Working with the front-end

When working with the front-end, we try limit our use of JavaScript as much as possible.
Actually, we are not even using JavaScript since our front-end also uses Rust thanks to WebAssembly.
But we want Plume to work with as little JavaScript as possible, since reading a post (Plume's first goal)
shouldn't require a lot of interactions with the page.

When editing SCSS files, it is good to know that they are compiled by `cargo` too. But `cargo` can be
a bit slow, since it recompiles all of Plume every time, not only the SCSS files. A workaround is to run
`cargo run` in the background, and use `cargo build` to compile your SCSS, then kill it before the end of
the build. To know when your SCSS have been compiled, wait for cargo to tell you it is compiling `plume(bin)`
and not `plume(build)` (next to the progress bar).

Also, templates are using the Ructe syntax, which is a mix of Rust and HTML. They are compiled to Rust
and embedded in Plume, which means you have to re-run `cargo` everytime you make a change to the templates.

# Code Style

For Rust, use the standard style. `rustfmt` can help you keeping your code clean.

For SCSS, the only rules are to use One True Brace Style and two spaces to indent code.

For JavaScript, we use [the JavaScript Standard Style](https://standardjs.com/).

For HTML/Ructe templates, we use HTML5 syntax.
