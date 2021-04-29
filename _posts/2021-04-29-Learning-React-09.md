---
layout: article
title: "[ #9 ] useEffect hook"
tags: React
---

# 1. useEffect

## useEffect란?

useEffect는 기본적으로 몇가지 조건에 의해 작동하게 됩니다. <br>

### 조건 1.

페이지가 처음 렌더링 되고난 후 useEffect 내의 함수는 무조건 한번 실행되고 봅니다.


~~~jsx
useEffect(() => {
    function doSomething(){
        console.log("something")
    }
    doSomething();
    
})
~~~
### 조건 2.
만약에 useEffect의 2번재 파라미터값이 없다면 이는 해당 컴포넌트가 렌더링 될 때마다 실행됩니다.
두번째 파라미터는 watch 값들의 목록입니다 만약 값이 변경되면 useEffect의 함수가 실행되기때문에 
아무것도 전달하지 않으면 계속실행됩니다.


~~~jsx
useEffect(() => {
    function doSomething(){
        console.log("something")
    }
    doSomething();
    
},[])
~~~

~~~jsx
useEffect(() => {
    function doSomething(){
        console.log(someProp);
    }
    doSomething();
    
},[someProp])
~~~
### 조건 3.

만약에 useEffect의 2번재 파라미터에 [] 빈 배열을 넣어주게되면 []를 지켜봐야하는 값으로 보게됩니다.
빈 배열은 컴포넌트가 랜더링되나 마나 변하지 않겠죠? 따라서 useEffect의 조건1인 컴포넌트의 첫 랜더링때 한번 실행되고 맙니다.

### clean-up을 이용하는 useEffect

네트워크에 request요청을 보내거나, DOM을 수동으로 조작하는것은 clean-up이 따로 필요 없습니다.
하지만 아래의 경우는 좀 다릅니다.
~~~jsx

function IntervalHookCounter() {
    const [count, setCount] = useState(0)
    
    const [stopwatch, setStopwatch] = useState(false)

    const tick = () => {
        setCount(count+1)
    }

    useEffect(() => {
        const interval = setInterval(tick, 1000)
        

    },[count])
~~~
해당코드는 얼핏 보면 잘 동작 할 것 같습니다만, 심각하게 문제가 있는 코드입니다.
문제는 interval에 있습니다. 랜더링 될 때마다 새로운 interval이 생성되기 때문입니다.
처음 몇초간은 타이머가 잘 동작하는것 같지만 조금지나면 시간이 올라가던 초가 고장나는게 보일 것입니다.
log 로 count를 찍어보면 이렇게 출력됩니다
1
1
2
1
2
3
1
2
3
4
1
2
3
4
5
따라서 인터벌의 청소가 필요한겁니다.
근데 어떻게 청소를할까요?

~~~jsx
useEffect(() => {
    const interval = setInterval(tick, 1000)
    return () => {
        clearInterval(interval)
        console.log(count);
    }
},[count])
~~~

간단합니다. useEffect함수 내에 함수를 return 하면 됩니다.
순서별로 보면 이렇습니다. <br>
<strong>
랜더링 =>
인터벌생성 =>
1초후 =>
인터벌 제거 =>
리 랜더링
</strong>
<br>
:star2: 리턴에있는 함수는 모든 작업이 끝나고, 랜더링 되기전에 실행됩니다.

---

useEffect에 친해지기위해 간단한 스톱워치를 만들어 보았습니다.
~~~jsx
import React, { useState, useEffect } from 'react'

function IntervalHookCounter() {
    //시간은 0초부터 시작합니다.
    const [count, setCount] = useState(0)
    //스톱워치는 처음에 멈춰있어요
    const [stopwatch, setStopwatch] = useState(false)

    
    useEffect(() => {
        
        let interval;
        //STOP watch 가 켜져 있을경우만 인터벌 생성해줍니다.
        if(stopwatch)
             interval = setInterval(() => {setCount(prev => prev + 1)}, 1000)
        return () => {
            clearInterval(interval)
            console.log("제거됨");
        }
        //count가 올라가거나, stop워치가 켜지거나 껴지는경우 return 함수가 실행되어 그다음 인터벌이 문제없이 생성되게 청소해줍니다.
    },[count,stopwatch])

    return (
        <div>
            {count}
            <button onClick={() => setStopwatch(prev => !prev)}>{stopwatch? 'OFF' : 'ON'}</button>
            <button onClick={() => setCount(0)}>Reset</button>
        </div>
    )
}
export default IntervalHookCounter

~~~