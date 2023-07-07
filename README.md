# Ostrich Editor

OstrichScript is a convention for extending nostr clients with user-defined functionality.
Ostrich Editor allows you to write, edit, publish, and delete these extensions.

Extensions are written in lua so that they are easy to write, and easy to sandbox in any client.
This editor uses [https://github.com/teoxoy/lua-in-js](lua-in-js) to compile and run scripts.
This particular implementation is very minimal, and is not a complete lua implementation, so
anything you write here should run anywhere.

You can learn lua by following along with [programming in lua](https://www.lua.org/pil/contents.html).

You can read more about OstrichScript [here](https://github.com/coracle-social/ostrich-script-js).

# Using the Editor

First, sign in using a NIP 07 extension. The editor will automatically query your kind 37077
events and show them in a list. From there, you can either edit existing scripts, or create a
new one using the button at the top.

When you're editing a script, there are two modes: editor and console.

Editor mode allows you to edit your script, including a title and description. As you work on your
script, Ostrich Editor will continuously compile it for you and show you any errors below.

When you're ready to test your script, you can switch to console mode, where you can write a script
in javascript using Ostrich Editor's own ndk and user. The return value of your script will be available
as `myScript`.
