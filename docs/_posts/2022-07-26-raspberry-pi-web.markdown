---
layout: posts
title:  "Raspberry Pi 4: Web Server Project"
author: Minjong Ha
published: false
date:   2022-07-26 13:34:00 +0900
---

## Abstract

## Installation

## Install node.js and npm

```bash
# as root
uname -m
armv7j

wget https://nodejs.org/dist/v16.16.0/node-v16.16.0-linux-armv7l.tar.xz
#https://nodejs.org/en/download/

tar -xvf node-v16.16.0-linux-armv7l.tar.xz

cd node-v16.16.0-linux-armv7l
cp -R * /usr/local
```

Above code represents the node.js and npm install in Raspberry Pi 4.
Other installation method, such as apt install, could be malfunction (I assume it is because Raspberry Pi has arm architecture).

```bash
node --version
npm --version
```

If you can see the versions of the packages, it successes.

```bash
npx create-react-app my-app # install react together
cd my-app
npm start

# access to http://${my_server_ip}:3000
```

You can create react application using above commands.
