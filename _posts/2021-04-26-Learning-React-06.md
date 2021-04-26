---
layout: article
title: "[ #6 ] setState & conditional content"
tags: React
---

# 1. useState 를 사용하는 방법.
## 기존코드
~~~jsx


const[expenses, setExpenses]=useState(initial_expenses)

  const addExpenseHandler = (expense) => {
    temp = [{
        ...data,
        expense,
    }]
    setExpenses(temp);
  }

~~~
함수 useState()는 현재값과, setExpenses라는 값을 변경하는 메소드를 반환하는줄 알았고, setExpenses(바꿀값) 을 통하여 
값을 직접 변경할 수 있는정도로만 알고있엇다. <br>

useState의 두번째 반환값 setExpenses을 활용하는 방법에 대해서 배웠다.<br>
만약에 currentState의 값이 prev값과 연관이 있을경우 이를 온전히 사용할 수 있다는것을 말이다.<br>
setExpenses 는 이전 state의 값을 가지고 있다는 것을 알 수 있었고, 값을 그냥 변경하는 경우가아닌 이전 값에 연관이 있을경우
아래의 코드와 같이 간결하게 표현 할 수 있는 방법을 배웠다.<br>
## 수정후
~~~jsx
const addExpenseHandler = (expense) => {
    setExpenses(prevExpense => {
      return [expense,...prevExpense]
    })
  }
~~~


---
# 2. Conditional content
무언가 보여줄때 조건문을 통하여 보여줄 수 있는 방법을 배웠다.
## 기존코드
~~~ jsx
    <Card className="expenses">
    {expenses}
    </Card>
    //만약에 expense에 아무 값이없을경우 아무것도 보이지 않아 허전하다.
~~~


## 조건문 적용후
~~~jsx
    <Card className="expenses">
    //삼항연산자를 이용하여 아무 값이 없을경우 없다고 출력해주고 존재한다면 해당 리스트 출력해주기.
    {expenses.length === 0 ? <p>No expenses found</p> : expenses}
    </Card>
~~~