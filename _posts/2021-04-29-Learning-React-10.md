---
layout: article
title: "[ #10 ] useContext hook"
tags: React
---

# useContext

## useContext란 무엇인가?

이때까지 부모 컴포넌트에서 자식컴포넌트에 정보를 전달할때 props를 통하여 전달 했었습니다.

A(부모) => B => C = > D 순으로 nested 되어있을경우 부모로부터 D로 데이터를 전송하려면 B, C, D 순으로 전달했었죠.

나중에 props의 이름을 까먹거나하면 정말 헷갈렸었는데 이번에 <strong>useContext</strong> 사용하는 법을 배우고 신세계가 열렸습니다.

:star2:useContext를 이용하면 B, C를 거치지 않고 A => D로 직행으로 원하는 데이터를 전송할 수 있습니다!
## useContext 어떻게 사용할까요? - 1

~~~jsx
import React from "react";

export const UserContext = React.createContext();
function App() {
  return (
    <UserContext.Provider value={"whatever you want to pass!"}>
      <div>wow</div>
    </UserContext.Provider>
  );
}

export default App;
~~~

### 1-1 React의 메서드의 하나인 createContext()를 이용하여 Context를 생성해줍니다.

### 1-2 생성된 Context 태그로 감싸줍니다.

### 1-3 원하는 정보를 value 안에 실어줍니다.

## useContext 어떻게 사용할까요? - 2

~~~jsx
import React,{useContext} from 'react'
import {UserContext} from '../App';


const DeeplyNestedComponent = () => {
    const message = useContext(UserContext);
    return (
        <div>
           {message}
           //wow whatever you want to pass!
        </div>
    )
}

export default DeeplyNestedComponent
~~~

### 2-1 정보보다 하위에 있으면서 정보를 전달할 컴포넌트.js 오픈해줍니다.

### 2-2 useContext hook 과  App.js에서 생성한 UserContext를 import 해줍니다.

### 2-3 원하는 이름의 variable에 useContext(UserContext)를 담아주고 필요한데로 정제하여 사용하시면 됩니다!

:star2:본 예제 에서는 간단히 스트링을 전달하였지만 배열이나 오브젝트 또한 전달이 가능합니다

