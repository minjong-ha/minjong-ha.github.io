---
layout: posts
title:  "Raspberry Pi 4: Web Server Project"
author: Minjong Ha
published: false
date:   2022-07-26 13:34:00 +0900
---

`React` and `node.js` deleverage the difficulty to deploy personal website.
You can use `react` to build frontend and `node.js` for backend.
In this post, I will describe my experience that implementing own website that shows summary of informations used in office.
Moreover, I use `ChatGPT-4` to implement most of the codes.

## Install node.js and npm

```bash
# as root
$ uname -m
armv7j

$ wget https://nodejs.org/dist/v16.16.0/node-v16.16.0-linux-armv7l.tar.xz
#https://nodejs.org/en/download/

$ tar -xvf node-v16.16.0-linux-armv7l.tar.xz

$ cd node-v16.16.0-linux-armv7l
$ cp -R * /usr/local
```

Above code represents the node.js and npm install in Raspberry Pi 4.

```bash
node --version
npm --version
```

If you can see the versions of the packages, it succeed.

```bash
npx create-react-app my-app # install react together
cd my-app
npm start

# access to http://${my_server_ip}:3000
```

You can create react application using above commands.

==================================================================

## Components and JSX

`npm start` command starts the react application by executing 'index.js'.

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
It is the part where JSX returns when index.js tries to use `<App />`.
Almost every components in this project will be returned as a JSX, except the database, computing, and etc.

You can access to `http://${YOUR_IP}:3000`.

=================================================================

## Moonphase

Since I like moonphase, I added moonphase on the top-center of the page.
Following is `MoonPhase.css` file that containing css data for moon and stars:

```javascript
//MoonPhase.css

@keyframes fillMoon {
    0% {
transform: scale(0);
    }
    100% {
transform: scale(1);
    }
}

@keyframes sparkle {
    0%, 100% {
opacity: 1;
    }
    50% {
opacity: 0.3;
    }
}

@keyframes moveRightToLeft {
    0% {
transform: translateX(0);
    }
    100% {
/*transform: translateX(-100%);*/
transform: translateX(-200%);
    }
}



.MoonPhase {
display: flex;
         justify-content: center;
         align-items: center;
         margin-top: 15px;
         margin-bottom: 15px;
}

.moon {
width: 100px;
height: 100px;
        border-radius: 50%;
        background-color: #e0e0e0; /* Slightly darker grey */
position: relative;
overflow: hidden;
border: 3px solid #ffffb1;
        box-sizing: border-box;
}

.moon-phase {
position: absolute;
top: 0;
left: 0;
width: 100%;
height: 100%;
        background-color: #333;
        border-radius: 50%;
        animation-name: fillMoon;
        animation-duration: 5s;
}

.StarsWrapper {
position: relative;
width: 100%;
height: 100%;
}

.StarsContainer {
position: absolute;
width: 100%;
height: 100%;
overflow: hidden;
          z-index: -1;
}


.star {
position: absolute;
width: 4px;
height: 4px;
        background-color: #ffffb1;
        border-radius: 50%;
animation: sparkle 2s linear infinite;
}
```

I used default css feature to represent the moon, and stars.
It also has `@keyframes` for animation that filling the moon and sparkling the stars.

And following is `MoonPhase.js` for moonphase:

```javascript
//MoonPhase.js

import React, { useState, useEffect } from 'react';
import './MoonPhase.css';

function MoonPhase() {
    const [moonPhase, setMoonPhase] = useState(0);

    useEffect(() => {
            const getMoonPhase = async () => {
            const apiKey = 'BSUPRTMGB5FSM5D8EBTU3ZYYS';
            // Coordinate for Seoul, Republic of Korea
            const lat = '37.5665';
            const lon = '126.9780';

            const response = await fetch(
                    `https://weather.visualcrossing.com/VisualCrossingWebServices/rest/services/timeline/${lat},${lon}/today?unitGroup=us&key=${apiKey}`
                    );
            const data = await response.json();
            const moonPhaseData = data.days[0].moonphase;

            setMoonPhase(moonPhaseData);
            };

            getMoonPhase();
            }, []);

    const phasePercentage = 100 - moonPhase * 100;

    useEffect(() => {
            // Start the animation when the component is mounted
            const moonPhaseEl = document.querySelector('.moon-phase');
            moonPhaseEl.style.animationPlayState = 'running';
            }, []);


    useEffect(() => {
            const starsContainer = document.querySelector(".StarsContainer");
            const numberOfStars = 50;

            for (let i = 0; i < numberOfStars; i++) {
            const star = document.createElement("div");
            star.classList.add("star");
            star.style.top = `${Math.random() * 100}%`;
            star.style.left = `${Math.random() * 100}%`;
            star.style.animationDelay = `${Math.random() * 2}s`; // Set random animation delays for the sparkle and moveRightToLeft animations
            star.style.animationDuration = `2s`; // Set random animation durations for the moveRightToLeft animation between 10 and 20 seconds


            starsContainer.appendChild(star);
            }
            }, []);

    return (
            <div className="StarsWrapper">
            <div className="StarsContainer"></div>
            <div className="MoonPhase">
            <div className="moon">
            <div
            className="moon-phase"
            style={{ clipPath: `inset(0% ${phasePercentage}% 0% 0%)` }}
            ></div>
            </div>
            </div>
            </div>
           );
}

export default MoonPhase;
```

And following is `App.js` that showing moonphase on its top:

```javascript
// App.js

import React, { useEffect } from 'react';
import './App.css';
import MoonPhase from './MoonPhase';

function App() {
    useEffect(() => {
            document.title = 'Dashboard';
            }, []);

    return (
            <div className="App">
            <MoonPhase />

            <header className="App-header">
            <h1>Dashboard</h1>
            </header>

            {/* Add the rest of your App component code here */}
            </div>
           );
}

export default App;

```

Since I want to make the sky looks like night, I added following codes on `App.css`:

```css
body {
    background-color: #2a2f4a; /* Dark blue */
margin: 0;
        font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Oxygen',
            'Ubuntu', 'Cantarell', 'Fira Sans', 'Droid Sans', 'Helvetica Neue',
            sans-serif;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
}
```

## Interesting Libraries

* [Moonphase Component](https://github.com/nikolas/react-moonphase)
* [Wolrd Time](https://github.com/prabhuignoto/react-worldtime)

## References

* [reactjs tutorial](https://ko.reactjs.org/tutorial/tutorial.html)
* [reactjs building web page posts](https://leftday.tistory.com/category/%EA%B0%9C%EB%B0%9C/react%20%ED%99%88%ED%8E%98%EC%9D%B4%EC%A7%80%20%EB%A7%8C%EB%93%A4%EA%B8%B0)
* [Message API (KakaoTalk)](https://velog.io/@da__hey/React-React-Typescript%EB%A5%BC-%ED%86%B5%ED%95%B4-%EC%B9%B4%EC%B9%B4%EC%98%A4%ED%86%A1-%EB%A9%94%EC%8B%9C%EC%A7%80-%ED%94%8C%EB%9E%AB%ED%8F%BC-API-%EC%9D%B4%EC%9A%A9%ED%95%B4%EB%B3%B4%EA%B8%B0)
* [Alpine.js?](https://alpinejs.dev/)
