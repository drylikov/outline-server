# Outline Server

[![Build Status](https://travis-ci.org/Jigsaw-Code/outline-server.svg?branch=master)](https://travis-ci.org/Jigsaw-Code/outline-server)

This repository has all the code needed to create and manage Outline servers on
DigitalOcean. An Outline server runs instances of Shadowsocks proxies and
provides an API used by the Outline Manager application.

Go to https://getoutline.org for ready-to-use versions of the software.

## Components

The system comprises the following components:

- **Outline Server**: a proxy server that runs a Shadowsocks instance for each
  access key and a REST API to manage the access keys. The Outline Server runs
  in a Docker container in the host machine.

  See [`src/shadowbox`](src/shadowbox)

- **Outline Manager:** an [Electron](https://electronjs.org/) application that
  can create Outline Servers on the cloud and talks to their access key
  management API to manage who has access to the server.

  See [`src/server_manager`](src/server_manager)

- **Metrics Server:** a REST service that the Outline Server talks to
  if the user opts-in to anonymous metrics sharing.

  See [`src/metrics_server`](src/metrics_server)


## Code Prerequisites

In order to build and run the code, you need the following installed:
  - [Node](https://nodejs.org/) (version `16.13.0`)
  - [NPM](http://npmjs.org/) (version `8.1.0`)
  - [Wine](https://www.winehq.org/download), if you would like to generate binaries for Windows.


> 💡 NOTE: if you have `nvm` installed, run `nvm use` to switch to the correct node version!

Install dependencies with:
```sh
npm install
```

This project uses [NPM workspaces](https://docs.npmjs.com/cli/v7/using-npm/workspaces/).


## Build System

We have a very simple build system based on package.json scripts that are called using `npm run`
and a thin wrapper for what we call build "actions".

We've defined a package.json script called `action` whose parameter is a relative path:
```shell
npm run action $ACTION
```

This command will define a `run_action()` function and call `${ACTION}.action.sh`, which must exist.
The called action script can use `run_action` to call its dependencies. The $ACTION parameter is
always resolved from the project root, regardless of the caller location.

The idea of `run_action` is to keep the build logic next to where the relevant code is.
It also defines two environmental variables:

- ROOT_DIR: the root directory of the project, as an absolute path.
- BUILD_DIR: where the build output should go, as an absolute path.

> ⚠️ To find all the actions in this project, run `npm run action:list`

### Build output

Building creates the following directories under `build/`:
- `web_app/`: The Manager web app.
  - `static/`: The standalone web app static files. This is what one deploys to a web server or runs with Electron.
- `electron_app/`: The launcher desktop Electron app
  - `static/`: The Manager Electron app to run with the electron command-line
  - `bundled/`: The Electron app bundled to run standalone on each platform
  - `packaged/`: The Electron app bundles packaged as single files for distribution
- `invite_page`: the Invite Page
  - `static`: The standalone static files to be deployed
- `shadowbox`: The Proxy Server

The directories have subdirectories for intermediate output:
- `ts/`: Autogenerated Typescript files
- `js/`: The output from compiling Typescript code
- `browserified/`: The output of browserifying the JavaScript code

To clean up:
```
npm run clean
```

## Shadowsocks Resistance Against Detection and Blocking

Shadowsocks used to be blocked in some countries, and because Outline uses Shadowsocks, there has been skepticism about Outline working in those countries. In fact, people have tried Outline in the past and had their servers blocked.

However, since the second half of 2020 things have changed. The Outline team and Shadowsocks community made a number of improvements that strengthened Shadowsocks beyond the censor's current capabilities.

As shown in the research [How China Detects and Blocks Shadowsocks](https://gfw.report/talks/imc20/en/), the censor uses active probing to detect Shadowsocks servers. The probing may be triggered by packet sniffing, but that's not how the servers are detected.

Even though Shadowsocks is a standard, it leaves a lot of room for choices on how it's implemented and deployed.

First of all, you **must use AEAD ciphers**. The old stream ciphers are easy to break and manipulate, exposing you to simple detection and decryption attacks. Outline has banned all stream ciphers, since people copy old examples to set up their servers. The Outline Manager goes further and picks the cipher for you, since users don't usually know how to choose a cipher, and it generates a long random secret, so you are not vulnerable to dictionary-based attacks.

Second, you need **probing resistance**. Both shadowsocks-libev and Outline have added that. The research [Detecting Probe-resistant Proxies](https://www.ndss-symposium.org/ndss-paper/detecting-probe-resistant-proxies/) showed that, in the past, an invalid byte would trigger different behaviors whether it was inserted in positions 49, 50 or 51 of the stream, which is very telling. That behavior is now gone, and the censor can no longer rely on that.

Third, you need **protection against replayed data**. Both shadowsocks-libev and Outline have added such protection, which you may need to enable explicitly on ss-libev, but it's the default on Outline.

Fourth, Outline and clients using shadowsocks-libev now **merge the SOCKS address and the initial data** in the same initial encrypted frame, making the size of the first packet variable. Before the first packet only had the SOCKS address, with a fixed size, and that was a giveaway.

The censors used to block Shadowsocks, but Shadowsocks has evolved, and as for 2021, it's ahead again in the cat and mouse game.

Shadowsocks remains our protocol of choice because it's simple, well understood and very performant. Furthermore, it has an enthusiastic community of very smart people behind it.
