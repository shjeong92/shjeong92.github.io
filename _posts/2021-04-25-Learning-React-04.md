---
layout: article
title: "[ #4 ] useState 사용하는 방법"
tags: React
---

# useState 를 사용하는 방법

## 방법 1. 여러개의 단일 useState 사용하기.
~~~jsx
const [enteredTitle, setEneteredTitle] = useState("");
const [enteredAmount, setEnteredAmount] = useState("");
const [enteredDate, setEnteredDate] = useState("");
const titleChangeHandler = (event) => {
setEneteredTitle(event.target.value);
};
const amountChangeHandler = (event) => {
    setEnteredAmount(event.target.value)
};
const dateChangeHandler = (event) => {
setEnteredDate(event.target.value)
};
~~~

---

## 방법 2. 여러개의 useState 묶어서 사용하기

~~~jsx
const [userInput, setUserInput] = useState({
    enteredTitle: '',
    enteredAmount: '',
    eneteredData: '',
});
~~~
객체 형식으로 모든 정보를 담아줍니다.<br>
userInput 또한 객체형태를 가지고 있을겁니다!


~~~jsx
const titleChangeHandler = (event) => {
    //setEneteredTitle(event.target.value);
    setUserInput({
        ...userInput,
        enteredTitle: event.target.value,
    });
};
const amountChangeHandler = (event) => {
    //setEnteredAmount(event.target.value)
    setUserInput({
        ...userInput,
        enteredAmount: event.target.value,
    });
};
const dateChangeHandler = (event) => {
    //setEnteredDate(event.target.value)
    setUserInput({
        ...userInput,
        eneteredDate: event.target.value,
    });
};
~~~
이번엔 각각의 메서드 부분입니다. 객체를 destructuring 하여 불러올때 원하는값만 불러올 수 있기에 원하는 값만 불러와도 되는줄 알았지만 리액트에서는 만약에 이전 값들을 불러오지 않는다면 그값은 없어지게됩니다. 따라서 '...userInput' 또한 넣어주어서 이전 state 들또한 모두 불러온뒤 필요한값을 수정해줍니다. <br>
어떻게 사용해야하는가에 대해서는 각자 원하는대로 하면 된다고 하는데 저는 오히려 따로따로 분리해서 쓰는게 편할거 같아서 필요한 상황이 오지않는다면 객체 형식으로 묶어서 쓰지는 않을 것 같습니다 .