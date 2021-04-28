---
layout: article
title: "[ #7 ] 3 ways to fullfill jsx's requirement"
tags: React
---

# JSX requirement

~~~jsx
import React from 'react'

const AddUser = () => {
  return (
    <div>이건</div>
    <div>에러나요</div>
  )
}

export default AddUser
~~~

리액트 jsx는 오직 하나의 html element만을 return할 수 있습니다. <br> 
따라서 위의 코드는 에러를 뿜어냅니다.

---

이를 수정하기 위해선 이 두태그를 하나의 div로 감싸줘야 합니다.

~~~jsx
import React from "react";

const AddUser = () => {
  return (
    <div>
      <div>이건</div>
      <div>에러안나요</div>>
    </div>
  );
};

export default AddUser;
~~~

하지만 여러가지 컴포넌트를 만들고 그것을 모두 <App> 에다가
떼려 박으면 실제로는 엄청나게 많은 필요없는 nested <div> 가 생기게 됩니다. <br>
그리고 무수히 많은 nested <div> 는 로딩 속도 저하에 한 몫 한다고 합니다. 따라서 이를 해결할 방법이 필요하겠죠?

## 해결방법 1.

Wrapper.js 라는 Helper component를 하나 만들어줍니다.
~~~jsx
import React from 'react'
const Wrapper = (props) => {
    return props.children;
}
export default Wrapper
~~~
Wrapper.js는 props.children 이라는 해당 컴포넌트의 모든 자식을 반환합니다. div로 감싸지 않더라도말이죠.

---

~~~jsx
import React from "react";
import Wrapper from './Wrapper'
const AddUser = () => {
  return (
    <Wrapper>
      <div>이것도</div>
      <div>에러안나요</div>>
    </Wrapper>
  );
};

export default AddUser;
~~~

이런식으로 Wrapper.js를 사용하면 div 없이도 모든 내용을 감쌀 수 있게됩니다! 

## 해결방법 2. 

~~~jsx
import React, {Fragment} from "react";
import Wrapper from './Wrapper'
const AddUser = () => {
  return (
    <Fragment>
      <div>이것도</div>
      <div>에러안나요</div>>
    </Fragment>
  );
};

export default AddUser;
~~~
커스텀 컴포넌트인 Wrapper를 사용하지 않아도 React에서 기본으로 제공하는 Fragment는
우리가 만들었던 Wrapper이란 똑같은 기능을 합니다.