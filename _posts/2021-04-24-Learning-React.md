---
layout: article
title: "[ Day - 1 ] Learning React "
tags: React
---
 
# 1. props
오늘은 props 에 대하여 배웠다.
custom component에 값들을 전송할때 해당 태그안에 객체이름=value식으로 보낼 수 있으며
해당 컴포넌트에서는 보내어진 모든 객체들을 하나의 객체안에 몽땅 넣어서 전달한다.

---
## App.js
ExpensesItem 으로 expenses 객체 전달하기.
~~~js
import ExpenseItems from "./components/ExpenseItems";
import './App.css';
function App() {
  const expenses = [
    {
      title: "Toilet Paper",
      amount: 94.67,
      date: new Date(2021, 2, 21),
    },
    {
      title: "New Tv",
      amount: 1294.67,
      date: new Date(2021, 2, 21),
    },
    {
      title: "Car insuarnce",
      amount: 294.67,
      date: new Date(2021, 2, 21),
    },
    {
      title: "Macbook pro",
      amount: 2294.67,
      date: new Date(2021, 2, 21),
    },
  ];
  return (
    <div className="main">
      <ExpenseItems expenses={expenses}/>
    </div>
  );
}

export default App;
~~~

---

## ExpenseItems.js
expenses를 통으로 보낼경우 객체 안-> 객체 안 -> 정보를 얻으므로 복잡함을 줄이기 위해 
expenses destructuring 하여 각각의 정보 개별 ExpenseItem으로 전달한다
~~~js
import React from "react";
import ExpenseItem from "./ExpenseItem";
function ExpenseItems(props) {
  const expenses = props.expenses.map(({ title, amount, date }) => {
    return <ExpenseItem title={title} amount={amount} date={date} />;
  });

  return <Exp className='expensesitem'>{expenses}</Exp>;
}

export default ExpenseItems;
~~~
---

## ExpenseItem.js
App -> ExpenseItems -> ExpensesItem 으로 전달받은 prop값으로 태그 찍어내주기
~~~js
import React from "react";
import ExpenseDate from "./ExpenseDate";
import "./ExpenseItem.css";
function ExpenseItem(props) {
  return (
    <div className="expense-item">
      <ExpenseDate date={props.date} />
      <div className="expense-item__description">
        <h2>{props.title}</h2>
        <div className="expense-item__price">${props.amount}</div>
      </div>
    </div>
  );
}

export default ExpenseItem;

~~~



---