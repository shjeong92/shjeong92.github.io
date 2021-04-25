---
layout: article
title: "[ #5 ] state의 역행 childretn -> parent"
tags: React
---

# 1. props의 역행
state는 top down 방식으로 이동 할 수 있는 줄 알았지만 아니었다. 부모 컴포넌트에서 함수를 선언한다음 자식 컴포넌트에 함수에대한 포인터를 전속하는 방식으로 자식 -> 부모에게 데이터 전송이 가능하다.

자세한 내용은 코드를통해 알아보도록한다.

~~~jsx 
import React, { useState } from "react";
import ExpenseItem from "./ExpenseItem";
import Card from "../UI/Card";
import ExpensesFilter from "./ExpensesFilter";
import "./Expenses.css";
const Expenses = (props) => {
  const [filteredYear, setFilteredYear] = useState('2020')

  const filterChangeHandler = (selectedYear) => {
    setFilteredYear(selectedYear);
    console.log(selectedYear);
  }
  const expenses = props.expenses.map(({ title, amount, date }) => {
    return <ExpenseItem title={title} amount={amount} date={date} />;
  });

  return (
    <div>
      <ExpensesFilter selected={filteredYear} onChangeFilter={filterChangeHandler} />
      <Card  className="expenses">{expenses}</Card>
    </div>
  );
};

export default Expenses;
~~~

Expenses.js 라는 컴포넌트에서 자식컴포넌트인 ExpensesFilter 컴포넌트에게 filterChangeHandler 함수의 포인터를 전송한다. 또한 초깃값으로 '2020'인 filteredYear도 전송한다.

~~~jsx

import React from 'react';

import './ExpensesFilter.css';

const ExpensesFilter = (props) => {
  const selectedYear = (evt) => {
    props.onChangeFilter(evt.target.value);
  }
  return (
    <div className='expenses-filter'>
      <div className='expenses-filter__control'>
        <label>Filter by year</label>
        <select value={props.selected} onChange={selectedYear}>
          <option value='2022'>2022</option>
          <option value='2021'>2021</option>
          <option value='2020'>2020</option>
          <option value='2019'>2019</option>
        </select>
      </div>
    </div>
  );
};
export default ExpensesFilter;

~~~

자식컴포넌트에선 select 태그가 initial value 인 2020을 가지고 있으며 onChange event를 추가하여 년도를 선택할떄마다 selectedYear 함수를 호출하는데, 해당 함수는 부모 컴포넌트의 함수를 가르키는 함수가 있다. <br>
이는 부모컴포넌트의 함수에 값을 넣고 실행하는데, 이러한 방식으로 자식->부모 간의 state정보의 역행이 가능한 것이다..

이러한 방식의 정보 전달은 가장 가까운 컴포넌트 한칸 한칸 직접이동하는 방법 이외에는 없다!! <br>
만약에 A 가 B 와 C의 컴포넌트이고,  B가 C의 정보가 필요하다면  C의정보를 A로 역행시킨후 그정보를 B로 전송 시키는 방식으로 전송하여야 한다.