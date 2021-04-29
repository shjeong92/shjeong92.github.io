---
layout: article
title: "[ #8 ] Portal, useRef"
tags: React
---

# 1. Portal을 사용하기.


~~~jsx
//index.js
import React from 'react';
import ReactDOM from 'react-dom';

import './index.css';
import App from './App';

ReactDOM.render(<App />, document.getElementById('root'));
~~~
modal, overlayed popup같은 경우는 따로 분리해서 보여주는게 맞다고 합니다. 하지만 어떻게 그렇게 할 수 있을까요?. 

~~~jsx
//index.html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <link rel="icon" href="%PUBLIC_URL%/favicon.ico" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="theme-color" content="#000000" />
    <meta
      name="description"
      content="Web site created using create-react-app"
    />
    <link rel="apple-touch-icon" href="%PUBLIC_URL%/logo192.png" />

    <link rel="manifest" href="%PUBLIC_URL%/manifest.json" />

    <title>React App</title>
  </head>
  <body>
    <noscript>You need to enable JavaScript to run this app.</noscript>
    // root 이외의 다른 div에 컴포넌트를 전달해봅시다.
    <div id="backdrop-root"></div>
    <div id="overlay-root"></div>
    <div id="root"></div>

  </body>
</html>

~~~

---

우리는 이때까지 모든 컴포넌트들을 App.js에 연결 시켜주었는데요 App.js는 index.js 는 App.js를 import하여, 이를 index.html의 body 안의 id가 root인 div안에 모든 내용을 보내는것과 같습니다.

하지만 에러메세지나 팝업같은경우 간단한 프로젝트의경우 그냥 똑같은 root div에 넣어도 상관이 없지만 큰 프로젝트의 경우 기존 레이아웃과의 충돌을 야기할 수 있어 다른 별개의 div에 출력하는게 좋다고 합니다.
이를 가능하게 해주는것이 <strong>Portal</strong> 입니다.



---
## portal 사용하는 방법.

### step 1.
~~~jsx
import ReactDom from 'react-dom';
~~~
ReactDom 을 import 해줍니다.

### step 2.
~~~jsx
const ErrorModal = (props) => {
  return (
    <Fragment>
      {ReactDom.createPortal(<Backdrop onConfirm={props.onConfirm}/>, document.getElementById('backdrop-root'))}
      {ReactDom.createPortal(<ModalOverlay title={props.title} message={props.message} onConfirm={props.onConfirm}/> ,document.getElementById('overlay-root'))}
    </Fragment>
  );
};
~~~
ReactDom의 createPortal메서드를 이용하여 원하는 div 에 원하는 컴포넌트를 연결시켜줄 수 있습니다.
createPortal(보내고자하는 컴포넌트와 props, 원하는 html내의 id)

inpect => element 를 확인해보면 정말 root div 이아닌 각각의 원하는 div에 컴포넌트들이 연결된 것을 확인 할 수 있습니다.


# 2.useRef 란 무엇일까요.
지금까지 useState만 써왔고 이를통해 계속해서 변하는 값을 불러올수도, 값을 set할 수도 있었죠.

useRef는 값을 읽어올때 주로 쓰입니다. 물론 useRef를 이용하여 값을 직접 수정 할 수도 있지만 권장하지 않는다고 합니다.
<br>

JavaScript 를 사용 할 때에는, 우리가 특정 DOM 을 선택해야 하는 상황에 getElementById, querySelector 같은 DOM Selector 함수를 사용해서 DOM 을 선택합니다.

React에서는 useRef()를 사용함으로써원하는 element의 정보를 모두 가져올 수 가 있습니다.

## useRef 사용하기

### step 1.

~~~jsx
import React, { useRef } from 'react';
~~~
useRef import 해주기.

### step 2.

~~~jsx
const nameInputRef = useRef();
const ageInputRef = useRef();
~~~

변수에 지정해주기

### step 3.

~~~jsx
    <input
    id="username"
    type="text"
    ref={nameInputRef}
    />
    <label htmlFor="age">Age (Years)</label>
    <input
    id="age"
    type="number"
    ref={ageInputRef}
    />
~~~
원하는 element에 ref값에 지정해준 ref 변수를 입력해줍니다.
nameInputRef.current은 username inputd모든 정보를 어디서든지 접근 할 수 있도록하여줍니다. 물론 값도 바꿀 수 있지만 권장되지는 않습니다.
따라서 값이 다이나믹하게 바뀌며 이를 읽거나 쓰는경우 useState를 사용하는게 좋고,
form 을통하여 입력된 값을 읽어서 무언가 처리해야 하거나 할대는 useRef를 사용하는게 불필요한 연산을 줄여 줄 수 있다고 합니다.