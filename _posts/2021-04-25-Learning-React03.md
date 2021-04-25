---
layout: article
title: "[ #3 ] useState 사용해보기"
tags: React
---

# Title을 바꿔봅시다.
## 왜 제목이 안바뀔까요~
~~~jsx
import React, { useState } from "react";
import ExpenseDate from "./ExpenseDate";
import Card from "../UI/Card";
import "./ExpenseItem.css";
const ExpenseItem = (props) => {
  const title = props.title;
  const clickHandler = () => {
    title = "Clicked!"
    console.log("You definitely clicked!");
  };
  return (
    <Card className="expense-item">
      <ExpenseDate date={props.date} />
      <div className="expense-item__description">
        <h2>{title}</h2>
        <div className="expense-item__price">${props.amount}</div>
      </div>
      <button onClick={clickHandler}>Change Title</button>
    </Card>
  );
}
export default ExpenseItem;
~~~

해당 코드를 보면,<br>
button 에 onclick 이벤트에 clickHandler 라는 함수를 가르쳐주었는데 이 함수는<br>
title 을 분명히 바꾸고있지만 타이틀은 바뀌지 않는다. 처음에는 정말 이해가 되지 않았다.<br>
콘솔로그를 보면 "You definitely clicked!" 는 계속 출력이 되지만 해당 Title은 변하지 않았기 때문이다.<br>
React에서는 기본적으로, 한번 rendering 되면 변수가 바뀌건 말건간에 그걸로 끝이다. 그렇기때문에 변수가 바뀌어도 보이지는 않는것이다.<br>

이를 다시 렌더링 하게하기 위해 사용되는것이 state 라고 한다. 

***

## 옳바른 코드
~~~jsx
import React, { useState } from "react";
import ExpenseDate from "./ExpenseDate";
import Card from "../UI/Card";
import "./ExpenseItem.css";
const ExpenseItem = (props) => {
  
  const [title, setTitle] = useState(props.title);
  const clickHandler = () => {
    setTitle('Updated!')
    console.log(title);
  };
  return (
    <Card className="expense-item">
      <ExpenseDate date={props.date} />
      <div className="expense-item__description">
        <h2>{title}</h2>
        <div className="expense-item__price">${props.amount}</div>
      </div>
      <button onClick={clickHandler}>Change Title</button>
    </Card>
  );
}

export default ExpenseItem;

해당코드는 버튼을 클릭하면 원하는대로 제목을 바꿔준다.

~~~jsx
const [title, setTitle] = useState(props.title);
~~~

useState메소드는 초기값을 parameter로 받으며, [title(초기값), setTitle(title을 바꾸는 메소드)]
를 반환한다. <br>
만약에 여기서 setTitle 메소드가 불리워지면 값을바꾸고 한번더 렌더링 해줌으로써 
우리가 직접 확인할 수 있게 되는 것이다.