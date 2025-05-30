---
title: "Firebase 로그인, 회원가입 with React Hook Form, yup"
date: "2024-03-19"
tags: ["스터디", "react-query", "firebase", "인증"]
desc: "구현 및 최적화 스터디 4"
---



## 1. 로그인, 회원가입 UI 구현

![alt text](/images/post-4/1.png)
![alt text](/images/post-4/2.png)

 
## 2. Firebase 연동
firebase 가입한 뒤, 프로젝트 생성해주고 기타 설정해준다.


```
//firebase.ts
// Import the functions you need from the SDKs you need
import { initializeApp } from "firebase/app";
import { getAnalytics } from "firebase/analytics";
// TODO: Add SDKs for Firebase products that you want to use
// https://firebase.google.com/docs/web/setup#available-libraries

// Your web app's Firebase configuration
// For Firebase JS SDK v7.20.0 and later, measurementId is optional
const firebaseConfig = {
  apiKey: `${process.env.REACT_APP_FIREBASE_API_KEY}`,
  authDomain: `${process.env.REACT_APP_FIREBASE_AUTH_DOMAIN}`,
  projectId: `${process.env.REACT_APP_FIREBASE_PROJECT_ID}`,
  storageBucket: `${process.env.REACT_APP_FIREBASE_STORAGE_BUCKET}`,
  messagingSenderId: `${process.env.REACT_APP_FIREBASE_SENDER_ID}`,
  appId: `${process.env.REACT_APP_FIREBASE_APP_ID}`,
  measurementId: `${process.env.REACT_APP_FIREBASE_MEASUREMENT_ID}`
};

// Initialize Firebase
const app = initializeApp(firebaseConfig);
const analytics = getAnalytics(app);

export default app;
```
 
## 3. 로그인, 회원가입 설정
로그인 방법은 이메일, Google, Github 3가지 방법으로 설정했다.

![alt text](/images/post-4/3.png)
 
<br />

useFirebaseLogin.ts hook 만들어서 firebase 로그인관련 코드는 따로 분리해주었다.


```
import { useState } from 'react';
import { getAuth, signInWithEmailAndPassword, GoogleAuthProvider, signInWithPopup, GithubAuthProvider, signOut } from 'firebase/auth';
import app from '../firebase';
import { useNavigate } from 'react-router-dom';

export const useFirebaseLogin = () => {
  const [error, setError] = useState('');
  const auth = getAuth(app);
  const navigate = useNavigate();

  const emailLogin = async (email: string, password: string) => {
    try {
      await signInWithEmailAndPassword(auth, email, password);
      navigate("/cat");
    } catch (error: any) {
      setError(error.message);
    }
  };

  const googleLogin = async () => {
    try {
      const provider = new GoogleAuthProvider();
      await signInWithPopup(auth, provider);
      navigate("/cat");
    } catch (error: any) {
      setError(error.message);
    }
  };

  const githubLogin = async () => {
    try {
      const provider = new GithubAuthProvider();
      await signInWithPopup(auth, provider);
      navigate("/cat");
    } catch (error: any) {
      setError(error.message);
    }
  };

  const logout = async () => {
    try {
      await signOut(auth);
    } catch (error: any) {
      setError(error.message);
    }
  };

  return { emailLogin, googleLogin, githubLogin, logout, error };
};
 ```


## 4. 로그인, 회원가입 유효성 검사
react-hook-form과 yup 라이브러리를 사용해서 유효성 검사를 구현했다.



```
import React from 'react';
import { useForm, Resolver } from 'react-hook-form';
import { yupResolver } from '@hookform/resolvers/yup';
import * as yup from 'yup';
import { Button, ErrorMessage, ID, Input, LoginContainer, LoginForm, PW, Account } from '../../styles/Login';
import { createUserWithEmailAndPassword, getAuth, updateProfile } from 'firebase/auth';
import app from '../../firebase';
import { Link, useNavigate } from 'react-router-dom';

interface FormType {
  username: string;
  email: string;
  password: string;
  checkPassword: string;
}

// yup 스키마
const schema = yup.object({
  username: yup.string().required('이름을 입력해주세요.'),
  email: yup.string().required('이메일을 입력해주세요.').email('유효하지 않은 이메일 형식입니다.'),
  password: yup.string().required('비밀번호를 입력해주세요.').min(6, '비밀번호는 최소 6자 이상이어야 합니다.'),
  checkPassword: yup.string().oneOf([yup.ref('password')], '비밀번호가 일치하지 않습니다.'),
}).required();

export default function SignUp() {
  const { register, handleSubmit, formState: { errors } } = useForm<FormType>({
    resolver: yupResolver(schema) as Resolver<FormType>,
    mode: 'all'
  });
  const [error, setError] = React.useState('');
  const auth = getAuth(app);
  const navigate = useNavigate();

  const onSubmit = async (data: FormType) => {
    const { username, email, password } = data;

    try {
      const userCredential = await createUserWithEmailAndPassword(auth, email, password);
      await updateProfile(userCredential.user, { displayName: username });
      navigate("/signin");
    } catch (error: any) {
      console.error(error);
      setError(error.message);
    }
  };

  return (
    <LoginContainer>
      <h2>회원가입</h2>
      <LoginForm onSubmit={handleSubmit(onSubmit)}>
        <div>
          <ID>Name</ID>
          <Input type="text" {...register('username')} />
          {errors.username && <ErrorMessage>{errors.username.message}</ErrorMessage>}

          <PW>Email</PW>
          <Input type="email" {...register('email')} />
          {errors.email && <ErrorMessage>{errors.email.message}</ErrorMessage>}

          <PW>Password</PW>
          <Input type="password" {...register('password')} />
          {errors.password && <ErrorMessage>{errors.password.message}</ErrorMessage>}

          <PW>Password Check</PW>
          <Input type="password" {...register('checkPassword')} />
          {errors.checkPassword && <ErrorMessage>{errors.checkPassword.message}</ErrorMessage>}
          {error && <ErrorMessage>{error}</ErrorMessage>}
        </div>
        <Button type="submit">Create Account</Button>
        <Account><Link to="/signin" style={{ color: "white" }}>Login</Link></Account>
      </LoginForm>
    </LoginContainer>
  );
}

```
 
![alt text](/images/post-4/4.png)
![alt text](/images/post-4/5.png)


## 5. 유저 정보 불러오기 - 이름, 프로필 사진

```
import React, { useState, useEffect, useMemo } from 'react';
import styled from "styled-components";
import { User, getAuth, onAuthStateChanged } from "firebase/auth";
import Search from '../components/Search';
import { FaRegUserCircle } from "react-icons/fa";
import { useNavigate } from 'react-router-dom';
import { useFirebaseLogin } from '../hooks/useFirebaseLogin';

export default function Header() {
  const navigate = useNavigate();
  
  //카테고리 관리
  const [shrink, setShrink] = useState(false);
  const handleScroll = () => {
    const currentScrollY = window.scrollY;
    setShrink(currentScrollY > 50);
  };
  useEffect(() => {
    window.addEventListener('scroll', handleScroll);
    return () => window.removeEventListener('scroll', handleScroll);
  }, []);

  //User
  const [user, setUser] = useState<User | null>(null); 
  const auth = getAuth();
  useEffect(() => {
    const userState = onAuthStateChanged(auth, (user) => {
      setUser(user);
    });
    return userState; 
  }, [auth]);
  const { logout } = useFirebaseLogin();

  return (
    <StyledHeader shrink={shrink}>
      <Search />
      <Profile>
        {user ? (
          <Username>
            {user.photoURL ? (
              <UserProfileImage src={user.photoURL} alt="User profile" />
            ) : (
              <FaRegUserCircle style={{marginRight: '5px'}}/>
            )}
            {user.displayName}
            <LogoutBtn onClick={() => logout()}>Logout</LogoutBtn>
          </Username> 
        ) : (
          <button onClick={() => navigate("/signin")}>로그인</button> 
        )}
      </Profile>
    </StyledHeader>
  );
}

```

![alt text](/images/post-4/6.png)