---
layout: article
title: "[ #2 ] Learning React "
tags: React
---

# 1. props.children & custom tag

컴포넌트를 호출할때 각각의 변수정보또한 같이 보내게 된다. 이는 하나의 객체안에 담겨서 넘어가게되는데
이때 children 이라는 정보든 따로 넘겨주지 않아도 기본적으로 전달되는 정보이다.

본글에서는 Card.js 라는 wrapper 컴포넌트를 만들면서 자세히 설명하도록 하겠다.


## Expenses.js
~~~jsx
import React from "react";
import ExpenseItem from "./ExpenseItem";
import Card from "../UI/Card"
import "./Expenses.css";
const Expenses = (props) => {
  //expenses의모든 내용을 각각의 expenseitem으로 만들어준다.
  const expenses = props.expenses.map(({ title, amount, date }) => {
    return <ExpenseItem title={title} amount={amount} date={date}/>;
  });
  //Card컴포넌트로 만들어진 expenseitem들을 감싸주고 className또한 지정해준다
  return <Card className="expenses">{expenses}</Card>;
}

export default Expenses;

~~~
우선 Card컴포넌트를 사용하는 Expenses.js를 자세히 보자. <br>
여기서 Card 컴포넌트는 기본 HTML태그가 아니므로 css의 영향을 받지 않고, className : expenses 라는 객체로 전달된다.


## Card.js

자, 이제 Card 컴포넌트를 살펴보자.

~~~jsx
import React from 'react'
import './Card.css';
const Card = (props) => {
    const classes = 'card '+ props.className;
    return (
        <div className={classes}>
            {props.children}
        </div>
    )
}

export default Card
~~~

따로 children 이라는 객체에대한 정보를 정해준적이 없지만 이미 가지고있다. children은 부모컴퍼넌트의 
open tag 와 close tag사이의 모든 태그정보들을 다 가지고 있다. 본 예제 에서는  expenses.js의 모든 expenseItem 즉,

~~~jsx
const expenses = props.expenses.map(({ title, amount, date }) => {
    return <ExpenseItem title={title} amount={amount} date={date}/>;
  });
~~~
Card 의 props.children 은 해당 정보를 모드 가지고 있는것이다. 
커스텀 컴포넌트에 className은 해당 컴포넌트에 바로 스타일적용이 되지 않지만 전송해주는 이유는 전달받은 이름을

~~~jsx
const classes = 'card '+ props.className;
return (
    <div className={classes}>
        {props.children}
    </div>
)
~~~
이런식으로 기본 HTML태그인 <div> 의 className에 넣어주어 스타일 적용이 가능하게 해줄 수 있기 때문이다.