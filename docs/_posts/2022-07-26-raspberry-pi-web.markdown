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

==================================================================

## Components and JSX

'npm-start' command starts the react application by executing 'index.js'.

Web page build by react has multiple components.
Each components can be represented as a chunk and the browser only deploys them.
We can implement the class component or functional component.

JSX is a extended javascript grammar.
It looks like html, but translated as a javascript.
Since we can write html at the same time, it has a high readability and easy to write.

We can check above features in the 'App.js'

```javascript
import logo from './logo.svg';
import './App.css';

function App() {
  return (
    <div className="App">
      <header className="App-header">
        <img src={logo} className="App-logo" alt="logo" />
        <p>
          Edit <code>src/App.js</code> and save to reload.
        </p>
        <a
          className="App-link"
          href="https://reactjs.org"
          target="_blank"
          rel="noopener noreferrer"
        >
          Learn React
        </a>
      </header>
    </div>
  );
}

export default App;

```

'App.js' defines App() function and we can see the html codes at return.
It is the part where JSX returns when index.js tries to use <App />.
Almost every components in this project will be returned as a JSX, except the database, computing, and etc.

=================================================================

## Interesting Libraries

* [Moonphase Component](https://github.com/nikolas/react-moonphase)
* [Wolrd Time](https://github.com/prabhuignoto/react-worldtime)

## References

* [reactjs tutorial](https://ko.reactjs.org/tutorial/tutorial.html)
* [reactjs building web page posts](https://leftday.tistory.com/category/%EA%B0%9C%EB%B0%9C/react%20%ED%99%88%ED%8E%98%EC%9D%B4%EC%A7%80%20%EB%A7%8C%EB%93%A4%EA%B8%B0)
* [Message API (KakaoTalk)](https://velog.io/@da__hey/React-React-Typescript%EB%A5%BC-%ED%86%B5%ED%95%B4-%EC%B9%B4%EC%B9%B4%EC%98%A4%ED%86%A1-%EB%A9%94%EC%8B%9C%EC%A7%80-%ED%94%8C%EB%9E%AB%ED%8F%BC-API-%EC%9D%B4%EC%9A%A9%ED%95%B4%EB%B3%B4%EA%B8%B0)
* [Alpine.js?](https://alpinejs.dev/)
